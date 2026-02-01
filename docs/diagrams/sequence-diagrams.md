# Sequence Diagrams

This document contains the key sequence diagrams showing message flows through the Communication Platform.

**Related Documents:**
- [Detailed Design](../detailed-design.md) — Component specifications
- [C4 Diagrams](./c4-diagrams.md) — Architecture views

---

## 1. Send Message Flow

The primary write path showing how a message flows from client to persistence and fan-out.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Gateway as API Gateway
    participant CmdSvc as Command Service
    participant NATS as NATS JetStream
    participant MsgWriter as Message Writer<br/>(Pull Consumer)
    participant Cassandra as Cassandra
    participant FanOut as Fan-Out Service<br/>(Pull Consumer)
    participant NotifSvc as Notification Service<br/>(Instance notif-07)
    participant OtherClient as Other Client(s)

    Client->>Gateway: POST /api/v1/chat.sendMessage<br/>{channel: "general", text: "Hello!"}
    Gateway->>Gateway: Validate JWT, Rate Limit
    Gateway->>CmdSvc: Forward request

    CmdSvc->>CmdSvc: Validate payload schema
    CmdSvc->>CmdSvc: Check authorization<br/>(cached permissions from NATS KV)
    CmdSvc->>CmdSvc: Generate event_id, message_id

    CmdSvc->>NATS: Publish to<br/>messages.send.ch_general<br/>(with Nats-Msg-Id for dedup)
    NATS-->>CmdSvc: +OK (persisted, replicated)

    CmdSvc-->>Gateway: HTTP 202 Accepted<br/>{message_id, correlation_id}
    Gateway-->>Client: HTTP 202 Accepted

    Note over NATS,MsgWriter: Async persistence (pull consumer)
    NATS->>MsgWriter: Fetch batch (pull)
    MsgWriter->>Cassandra: INSERT INTO messages (...)
    Cassandra-->>MsgWriter: OK
    MsgWriter->>NATS: +ACK

    Note over NATS,FanOut: Fan-Out routing (pull consumer)
    NATS->>FanOut: Fetch batch (pull)
    FanOut->>FanOut: Lookup routing table:<br/>ch_general → {notif-01, notif-07, notif-12}
    FanOut->>NotifSvc: Core NATS: instance.events.notif-07<br/>{type:"message.new", channel:"general", ...}
    Note right of FanOut: Also publishes to notif-01, notif-12

    NotifSvc->>NotifSvc: Local routing:<br/>ch_general → [ws_alice, ws_bob]
    NotifSvc->>OtherClient: WebSocket push:<br/>{"type":"message.new",<br/>"channel_id":"ch_general",<br/>"message":{...}}
```

### Key Properties

1. **Non-blocking write:** Client receives `202 Accepted` as soon as NATS acknowledges (~5ms)
2. **Guaranteed persistence:** NATS RAFT replication (R=3) ensures durability before ACK
3. **Instance-level fan-out:** One NATS publish per instance, not per user
4. **Idempotent writes:** `Nats-Msg-Id` enables publisher deduplication

---

## 2. User Connection & Subscription Flow

Shows what happens when a user connects via WebSocket, including presence and routing table updates.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client (Alice)
    participant NotifSvc as Notification Service<br/>(Instance notif-07)
    participant NATSKV as NATS KV Store
    participant FanOut as Fan-Out Service
    participant MongoDB as MongoDB

    Note over Client,NotifSvc: Connection Establishment
    Client->>NotifSvc: WebSocket Connect + Auth Token
    NotifSvc->>NotifSvc: Validate JWT, extract user_id

    NotifSvc->>MongoDB: Query channel memberships<br/>for user alice (cached)
    MongoDB-->>NotifSvc: [ch_general, ch_dev, ch_random, ... 99,997 more]

    NotifSvc->>NotifSvc: Build local routing table:<br/>for each channel_id, add alice's WS conn

    NotifSvc->>NATSKV: PUT presence/user/alice<br/>{"status":"online",<br/>"instance_id":"notif-07"}

    NotifSvc-->>Client: {"type":"auth.ok",<br/>"user_id":"usr_alice",<br/>"session_id":"sess_01HZ..."}

    Note over NATSKV,FanOut: Routing Table Update
    FanOut->>NATSKV: Watch("user/>") receives change
    FanOut->>FanOut: Load alice's channel memberships<br/>(from internal cache or MongoDB)
    FanOut->>FanOut: For each of alice's 100K channels:<br/>add notif-07 to routing table entry

    Note over Client,NotifSvc: Alice is now receiving<br/>real-time events via<br/>notif-07's instance inbox.<br/>Zero per-channel NATS<br/>subscriptions were created.

    Note over Client,NotifSvc: Disconnection
    Client->>NotifSvc: WebSocket Close
    NotifSvc->>NotifSvc: Remove alice from local routing table
    NotifSvc->>NATSKV: PUT presence/user/alice<br/>{"status":"offline",<br/>"last_seen":"2026-02-01T11:00:00Z"}
    FanOut->>NATSKV: Watch receives offline event
    FanOut->>FanOut: Remove alice from routing table<br/>(evict notif-07 from channels<br/>where alice was the only user<br/>on that instance)
```

### Key Properties

1. **Zero per-channel subscriptions:** 100K memberships = zero NATS subscriptions created
2. **In-memory routing update:** ~5-10ms for 100K channel insertions
3. **Graceful crash handling:** KV TTL (120s) auto-expires stale presence

---

## 3. Typing Indicator Flow (Ephemeral)

Shows how typing indicators flow without persistence overhead.

```mermaid
sequenceDiagram
    autonumber
    participant Alice as Alice (on notif-07)
    participant Notif07 as Notification Service<br/>(Instance notif-07)
    participant NATS as NATS Core Pub/Sub
    participant FanOut as Fan-Out Service
    participant Notif01 as Notification Service<br/>(Instance notif-01)
    participant Bob as Bob (on notif-01)

    Alice->>Notif07: WS: {"type":"typing.start",<br/>"channel_id":"ch_general"}
    Notif07->>NATS: Publish client.typing.ch_general<br/>{"user_id":"alice","typing":true}

    NATS->>FanOut: Deliver (subscribed to client.typing.>)
    FanOut->>FanOut: Lookup ch_general:<br/>{notif-01, notif-07, notif-12}
    FanOut->>Notif01: Core NATS: instance.events.notif-01<br/>{type:"typing", ...}
    Note right of FanOut: Skip notif-07 (originator).<br/>Also publish to notif-12.

    Notif01->>Notif01: Local routing: ch_general → [ws_bob, ws_carol]
    Notif01->>Bob: WS: {"type":"typing",<br/>"channel_id":"ch_general",<br/>"user_id":"alice","typing":true}
```

### Key Properties

1. **No persistence:** Core NATS pub/sub, not JetStream
2. **Origin exclusion:** Originating instance is skipped
3. **Same routing path:** Uses channel → instance lookup

---

## 4. Thread Reply Flow

Shows dual-write strategy for thread messages.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant CmdSvc as Command Service
    participant NATS as NATS JetStream
    participant MsgWriter as Message Writer
    participant Cassandra as Cassandra
    participant FanOut as Fan-Out Service
    participant NotifSvc as Notification Service

    Client->>CmdSvc: POST /api/v1/chat.sendMessage<br/>{channel: "general", thread_id: "msg_123", text: "Reply!"}

    CmdSvc->>CmdSvc: Validate thread exists<br/>Add thread metadata to event
    CmdSvc->>NATS: Publish messages.send.ch_general<br/>{..., thread_id: "msg_123", also_send_to_channel: false}

    NATS->>MsgWriter: Fetch batch

    Note over MsgWriter,Cassandra: Dual Write (parallel)
    par Thread table write
        MsgWriter->>Cassandra: INSERT INTO thread_messages<br/>(thread_id, bucket, message_id, ...)
    and Summary update
        MsgWriter->>Cassandra: UPDATE thread_summary<br/>SET reply_count = reply_count + 1<br/>WHERE thread_id = 'msg_123'
    end

    Note over MsgWriter: No write to messages table<br/>(also_send_to_channel: false)

    MsgWriter->>NATS: +ACK

    NATS->>FanOut: Fetch batch
    FanOut->>FanOut: Lookup thread subscribers:<br/>ch_general + thread followers
    FanOut->>NotifSvc: instance.events.{id}<br/>{type:"thread.reply", ...}

    NotifSvc->>Client: WebSocket: thread.reply event
```

### Thread Write Scenarios

| Scenario | `messages` table | `thread_messages` table |
|----------|------------------|------------------------|
| Thread-only reply | No | Yes |
| Also send to channel | Yes | Yes |

---

## 5. Client Reconnection Flow

Shows tiered catchup strategy based on disconnection duration.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant NotifSvc as Notification Service
    participant NATSKV as NATS KV
    participant NATSJS as NATS JetStream
    participant QuerySvc as Query Service
    participant Cassandra as Cassandra

    Client->>NotifSvc: WebSocket Connect<br/>{last_event_seq: 12345, last_seen: "2min ago"}
    NotifSvc->>NotifSvc: Calculate gap duration

    alt Tier 1: Gap < 2 minutes
        NotifSvc->>NATSJS: Create ephemeral consumer<br/>DeliverByStartSequence(12345)
        NATSJS-->>NotifSvc: Replay ~200-2000 events
        NotifSvc->>Client: Stream missed events via WebSocket
    else Tier 2: Gap 2min - 1hr
        NotifSvc->>NATSKV: GET client-state/{user_id}
        NATSKV-->>NotifSvc: {active_channels: [top 50]}
        NotifSvc->>Cassandra: Query messages for active channels<br/>(parallel, ~50 queries)
        Cassandra-->>NotifSvc: Recent messages
        NotifSvc->>Client: Catchup payload + unread counts
    else Tier 3: Gap 1hr - 24hr
        NotifSvc->>NATSKV: Batch GET read-pointer/* + channel-latest/*
        NATSKV-->>NotifSvc: Pointers for all channels
        NotifSvc->>NotifSvc: Compute unread counts only
        NotifSvc->>Client: Unread counts (no messages)
        Note right of Client: Client fetches messages<br/>via REST on channel open
    else Tier 4: Gap > 24hr
        NotifSvc->>Client: {"type":"full_refresh_required"}
        Client->>QuerySvc: GET /api/v1/channels.list
        QuerySvc-->>Client: Full channel list + metadata
    end
```

### Tier Summary

| Tier | Gap Duration | Data Source | Latency |
|------|-------------|-------------|---------|
| 1 | < 2 min | JetStream replay | < 1s |
| 2 | 2 min - 1 hr | Cassandra (active channels) | ~500ms |
| 3 | 1 hr - 24 hr | NATS KV (unread only) | < 100ms |
| 4 | > 24 hr | Full REST refresh | Varies |

---

## 6. Read Receipt Flow

Shows hybrid receipt handling based on channel size.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Alice (reader)
    participant NotifSvc as Notification Service
    participant CmdSvc as Command Service
    participant NATS as NATS JetStream
    participant ReceiptWriter as Receipt Writer
    participant Cassandra as Cassandra
    participant NATSKV as NATS KV
    participant OtherClient as Bob (sender)

    Client->>NotifSvc: WS: {"type":"message.read",<br/>"channel_id":"ch_dm_alice_bob",<br/>"message_id":"msg_456"}

    NotifSvc->>CmdSvc: Forward read event
    CmdSvc->>CmdSvc: Check channel_member_count (from event metadata)

    alt Small channel (≤50 members)
        CmdSvc->>NATS: Publish receipts.read.ch_dm_alice_bob
        NATS->>ReceiptWriter: Fetch batch
        ReceiptWriter->>Cassandra: INSERT INTO message_receipts<br/>(channel_id, message_id, user_id, read_at)
        ReceiptWriter->>NATS: +ACK

        Note over NATS,OtherClient: Fan-out read receipt
        NATS->>NotifSvc: Fan-out: receipt.read event
        NotifSvc->>OtherClient: WS: {"type":"receipt.read",<br/>"message_id":"msg_456",<br/>"read_by":["alice"]}
    else Large channel (>50 members)
        CmdSvc->>NATSKV: PUT read-pointer/alice/ch_large<br/>{"last_read":"msg_789"}
        Note right of NATSKV: No per-message receipt stored.<br/>No real-time fan-out.
    end
```

### Receipt Threshold

| Channel Type | Members | Storage | Real-time Update |
|--------------|---------|---------|------------------|
| DM | 2 | Per-message | Yes |
| Small group | ≤50 | Per-message | Yes |
| Large channel | >50 | Pointer only | No |
