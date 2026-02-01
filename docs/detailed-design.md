# Detailed Design

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01
**Version:** 3.0

---

## Table of Contents

1. [Service Specifications](#1-service-specifications)
   - [API Gateway](#11-api-gateway)
   - [Command Service](#12-command-service)
   - [Query Service](#13-query-service)
   - [Fan-Out Service](#14-fan-out-service)
   - [Notification Service](#15-notification-service)
   - [Auth Service](#16-auth-service)
2. [Worker Pool Specifications](#2-worker-pool-specifications)
   - [Message Writer Worker](#21-message-writer-worker)
   - [Meta Writer Worker](#22-meta-writer-worker)
   - [Search Indexer Worker](#23-search-indexer-worker)
   - [Push Notification Worker](#24-push-notification-worker)
   - [Webhook Dispatcher Worker](#25-webhook-dispatcher-worker)
3. [Webhook System Design](#3-webhook-system-design)
4. [NATS JetStream Topology](#4-nats-jetstream-topology)
5. [Redis/Valkey Usage](#5-redisvalkey-usage)
6. [Subscription & Fan-Out Architecture](#6-subscription--fan-out-architecture)
7. [Event Replay and State Reconstruction](#7-event-replay-and-state-reconstruction)
8. [Technical Considerations](#8-technical-considerations)
9. [Multi-Region Deployment Topology](#9-multi-region-deployment-topology)

---

## 1. Service Specifications

### 1.1 API Gateway

**Technology:** Envoy Proxy (or Nginx with lua-resty modules)

**Responsibilities:**

- TLS termination
- JWT token validation (stateless, using a shared public key)
- Rate limiting (per-user, per-IP)
- Route splitting:
  - `POST/PUT/DELETE` → Command Service
  - `GET` → Query Service
  - WebSocket upgrades → Notification Service
- Health check endpoints for all downstream services
- Request ID injection (for distributed tracing with OpenTelemetry)

**Scaling:** Stateless — scale horizontally behind a load balancer.

---

### 1.2 Command Service (Write Side)

**Technology:** Go

**Responsibilities:**

1. Accept write requests (send message, create channel, update profile, etc.)
2. Validate input (schema validation, authorization checks against cached permissions)
3. **Preprocess messages:** Parse markdown, extract mentions/links, validate adaptive cards, resolve emoji
4. Route by message type: regular messages → MESSAGES stream, ephemeral → EPHEMERAL stream
5. Publish a domain event to the appropriate NATS JetStream stream
6. Return an immediate acknowledgment to the client (HTTP 202 Accepted) with a correlation ID

**Key Design Principle:** The Command Service does **not** write directly to any database. It validates, preprocesses, and publishes. This is the fundamental decoupling from the old Meteor architecture.

#### Message Preprocessing Pipeline

```
┌──────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────┐
│ Validate │──►│ Preprocess   │──►│ Route by     │──►│ Publish  │
│ Request  │   │ Message      │   │ Message Type │   │ to NATS  │
└──────────┘   └──────────────┘   └──────────────┘   └──────────┘
                     │                   │
                     ▼                   ▼
              • Parse markdown     • Plain/Rich → MESSAGES
              • Extract @mentions  • Ephemeral → EPHEMERAL
              • Extract URLs       • Card → validate, MESSAGES
              • Validate cards
              • Resolve :emoji:
```

See [Message Formats](./features/message-formats.md) for detailed preprocessing specifications.

#### NATS Subject Hierarchy

```
messages.send.{channel_id}         → MessageSent event
messages.edit.{channel_id}         → MessageEdited event
messages.delete.{channel_id}       → MessageDeleted event
messages.react.{channel_id}        → ReactionAdded event
messages.thread_reply.{channel_id} → ThreadReply event
channels.create                    → ChannelCreated event
channels.update.{channel_id}       → ChannelUpdated event
channels.member.join.{channel_id}  → MemberJoined event
channels.member.leave.{channel_id} → MemberLeft event
users.update.{user_id}             → UserUpdated event
users.status.{user_id}             → StatusChanged event
threads.follow.{channel_id}        → ThreadFollowed / ThreadUnfollowed event
webhooks.outgoing.{channel_id}     → WebhookTrigger event
```

#### Event Envelope Schema (JSON)

```json
{
  "event_id": "evt_01HZ3K4M5N6P7Q8R9S0T",
  "event_type": "message.sent",
  "timestamp": "2026-02-01T10:30:00.123Z",
  "correlation_id": "req_abc123",
  "actor": {
    "user_id": "usr_xyz789",
    "username": "alice"
  },
  "payload": {
    "message_id": "msg_01HZ3K...",
    "channel_id": "ch_general",
    "text": "Hello, world!",
    "attachments": [],
    "thread_id": null
  },
  "metadata": {
    "client_ip": "192.168.1.1",
    "user_agent": "RocketChat/5.0",
    "trace_id": "otel_trace_abc"
  }
}
```

**Idempotency:** The Command Service generates a deterministic `event_id` from the request. NATS JetStream's publish deduplication (using `Nats-Msg-Id` header) ensures that retried publishes do not create duplicate events.

**Scaling:** Stateless — any number of instances can run behind the gateway.

---

### 1.3 Query Service (Read Side)

**Technology:** Go

**Responsibilities:**

- Serve all `GET` requests from clients: message history, channel list, user profiles, search results
- Read from the **appropriate data store** for each query type:
  - Message history → Cassandra
  - User/channel metadata → MongoDB
  - Full-text search → Elasticsearch
- Implement cursor-based pagination for message history (using `message_id` as the cursor)
- Cache frequently accessed data (channel membership lists, user profiles) using Redis/Valkey or a local TTL cache

**API Compatibility:** The Query Service implements existing REST API endpoints (e.g., `/api/v1/channels.history`, `/api/v1/channels.list`, `/api/v1/users.info`). Clients do not need to change their REST usage.

**Scaling:** Stateless — reads can be load-balanced across any number of instances.

---

### 1.4 Fan-Out Service (Channel → Instance Router)

**Technology:** Go

**This is the central component that solves the subscription explosion problem.**

**Responsibilities:**

1. Consume channel events from JetStream (messages, typing, presence, membership changes)
2. Maintain an in-memory **routing table** that maps each channel to the set of Notification Service instances with online members in that channel
3. For each channel event, look up the routing table and publish the event to the inbox of each relevant Notification Service instance — **not** to each individual user
4. Track membership changes (join/leave) and presence changes (online/offline) to keep the routing table current

**Why this exists:** Without the Fan-Out Service, each Notification Service instance would need one NATS subscription per channel per connected user. A user in 100K channels on an instance with 10K users creates up to 1 billion subscription evaluations. The Fan-Out Service collapses this to ONE subscription per Notification Service instance.

#### NATS Consumers

```
Consumer: fan-out-pool
  Stream: MESSAGES
  Type: Pull, Durable
  Filter: messages.>
  Ack Policy: Explicit
  Max Ack Pending: 5000

Consumer: fan-out-meta
  Stream: META
  Type: Pull, Durable
  Filter: channels.member.>
  Ack Policy: Explicit
```

**Scaling:** Multiple Fan-Out Worker instances bind to the same durable pull consumer. NATS distributes messages across them automatically. Each instance maintains a full copy of the routing table (~2GB). Since each message is processed by exactly one worker and the routing table is read-only during fan-out, no cross-worker coordination is needed.

---

### 1.5 Notification Service (WebSocket Termination & Local Routing)

**Technology:** Go (goroutines for high-concurrency WebSocket management)

**Responsibilities:**

1. Manage persistent WebSocket connections from clients
2. Authenticate each connection and associate it with a user
3. Subscribe to a **single NATS Core subject**: `instance.events.{self.instance_id}`
4. On receiving an event from the Fan-Out Service via the instance inbox, route it to the appropriate local WebSocket connections based on channel membership
5. Handle outbound client events (typing indicators, etc.) by publishing to NATS for the Fan-Out Service to redistribute
6. Publish presence changes (user online/offline) to Redis on connect/disconnect

**Critical difference from traditional design:** The Notification Service does NOT subscribe to per-channel NATS subjects. It subscribes to exactly ONE subject — its own instance inbox.

#### Internal State (per instance)

```
Local Routing Table:
  channel_id → Set<*WebSocketConn>

Connection Registry:
  user_id    → *WebSocketConn
  conn_id    → {user_id, Set<channel_id>}
```

When a user connects, the instance loads their channel memberships (from cache or MongoDB) and populates the local routing table. When an event for channel X arrives via the instance inbox, the instance looks up `channel_id → Set<*WebSocketConn>` and pushes to each connection. This is a pure in-memory operation with no NATS overhead.

#### WebSocket Protocol (JSON)

**Client → Server:**

```json
{"type": "auth", "token": "eyJhbG..."}
{"type": "ping"}
{"type": "typing.start", "channel_id": "ch_general"}
{"type": "typing.stop", "channel_id": "ch_general"}
{"type": "mark_read", "channel_id": "ch_general", "message_id": "msg_01HZ..."}
{"type": "channel.focus", "channel_id": "ch_general"}
{"type": "channel.blur", "channel_id": "ch_general"}
```

**Server → Client:**

```json
{"type": "auth.ok", "user_id": "usr_alice", "session_id": "sess_01HZ..."}
{"type": "pong"}
{"type": "message.new", "channel_id": "ch_general", "message": {...}}
{"type": "message.edited", "channel_id": "ch_general", "message_id": "msg_01HZ...", "text": "Hello, edited!", "edited_at": "..."}
{"type": "message.deleted", "channel_id": "ch_general", "message_id": "msg_01HZ..."}
{"type": "typing", "channel_id": "ch_general", "user_id": "usr_bob", "typing": true}
{"type": "presence", "user_id": "usr_bob", "status": "online"}
{"type": "member.joined", "channel_id": "ch_general", "user_id": "usr_carol"}
{"type": "member.left", "channel_id": "ch_general", "user_id": "usr_carol"}
{"type": "thread.reply", "channel_id": "ch_general", "thread_id": "msg_01HZ3K...", "message": {...}}
{"type": "thread.updated", "channel_id": "ch_general", "thread_id": "msg_01HZ3K...", "reply_count": 42, "last_reply_at": "..."}
{"type": "receipt.read", "channel_id": "dm_alice_bob", "message_id": "msg_01HZ...", "user_id": "usr_alice", "read_at": "..."}
```

**Scaling:** Each instance handles 10,000–50,000 WebSocket connections. The gateway routes new connections using least-connections load balancing. Adding instances does not increase NATS subscription count.

---

### 1.6 Auth Service

**Technology:** Go

**Responsibilities:**

1. Handle OIDC authentication flows (Authorization Code + PKCE)
2. Exchange authorization codes for tokens with OIDC provider
3. Manage refresh tokens (encrypted storage in Redis)
4. Create and validate user sessions
5. Provision users on first login (JIT provisioning)
6. Generate one-time WebSocket authentication tokens
7. Handle logout flows (local and OIDC single sign-out)

**Key Design Principle:** The Auth Service is the only component that communicates with the OIDC provider. All other services rely on JWT tokens validated at the API Gateway.

#### Authentication Flow

```
┌──────────┐                    ┌──────────────┐                    ┌─────────────┐
│  Client  │                    │ Auth Service │                    │    OIDC     │
└────┬─────┘                    └──────┬───────┘                    └──────┬──────┘
     │ 1. GET /auth/login              │                                   │
     │────────────────────────────────►│                                   │
     │                                 │ 2. Store state + PKCE in Redis    │
     │ 3. Redirect to OIDC             │                                   │
     │◄────────────────────────────────│                                   │
     │ 4. User authenticates           │                                   │
     │──────────────────────────────────────────────────────────────────────►
     │ 5. Redirect with auth code      │                                   │
     │◄──────────────────────────────────────────────────────────────────────
     │ 6. GET /auth/callback           │                                   │
     │────────────────────────────────►│                                   │
     │                                 │ 7. Exchange code for tokens       │
     │                                 │──────────────────────────────────►│
     │                                 │◄──────────────────────────────────│
     │                                 │ 8. Provision/update user          │
     │                                 │ 9. Create session                 │
     │ 10. Session cookie + access     │                                   │
     │◄────────────────────────────────│                                   │
```

#### Token Management

| Token | Lifetime | Storage | Purpose |
|-------|----------|---------|---------|
| **Access Token** | 15 min | Client memory | JWT for API authentication |
| **Refresh Token** | 7 days | Redis (encrypted) | Obtain new access tokens |
| **Session Token** | 7 days | HTTP-only cookie | Identify session for refresh |
| **WS Auth Token** | 30 sec | Redis | One-time WebSocket auth |

#### API Endpoints

```
POST /auth/login       → Initiate OIDC login, return auth URL
GET  /auth/callback    → Handle OIDC callback, exchange code
POST /auth/refresh     → Refresh access token (uses session cookie)
POST /auth/ws-token    → Generate one-time WebSocket token
POST /auth/logout      → Local logout (terminates session)
POST /auth/logout/all  → Logout all devices
GET  /auth/sessions    → List active sessions for user
DELETE /auth/sessions/{id} → Revoke specific session
```

**Scaling:** Stateless — any instance can handle requests. Session state stored in Redis.

See [Authentication Feature Documentation](./features/authentication.md) for detailed specifications.

---

## 2. Worker Pool Specifications

Workers are **pull-based JetStream consumers** — they request batches of messages, process them, and acknowledge.

### 2.1 Message Writer Worker

**Technology:** Go
**NATS Consumer:** Durable pull consumer on stream `MESSAGES`, consumer name `msg-writer-pool`
**Responsibility:** Persist message events to Apache Cassandra

```
Stream: MESSAGES
  Subjects: messages.>
  Retention: Limits (max age: 30 days, max bytes: configurable)
  Replicas: 3
  Storage: File

Consumer: msg-writer-pool
  Type: Pull (shared across N worker instances)
  Ack Policy: Explicit
  Max Ack Pending: 1000
  Max Deliver: 5 (with exponential backoff)
  Filter: messages.send.>, messages.edit.>, messages.delete.>, messages.thread_reply.>
```

**Batch Processing:** Workers call `Fetch(batch=50, timeout=5s)`, process the batch, write to Cassandra, and acknowledge. If a write fails, the message is NACK'd and redelivered.

#### Cassandra Schema: Messages

```sql
CREATE TABLE messages (
    channel_id    text,
    bucket        text,        -- time bucket: YYYY-MM-DD (partition sharding)
    message_id    timeuuid,    -- TimeUUID for natural ordering
    sender_id     text,
    body          text,
    attachments   list<frozen<attachment_type>>,
    thread_id     text,
    edited_at     timestamp,
    deleted       boolean,
    reactions     map<text, frozen<set<text>>>,
    created_at    timestamp,
    PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy',
                    'compaction_window_size': 1,
                    'compaction_window_unit': 'DAYS'}
  AND default_time_to_live = 0;
```

**Partition Strategy:** `(channel_id, bucket)` where `bucket` is a daily date string (e.g., `2026-02-01`). This prevents unbounded partition growth while keeping time-range queries efficient.

---

### 2.2 Meta Writer Worker

**Technology:** Go
**NATS Consumer:** Durable pull consumer on stream `META`, consumer name `meta-writer-pool`
**Responsibility:** Persist channel, user, and permission changes to MongoDB

```
Stream: META
  Subjects: channels.>, users.>
  Retention: Limits (max age: 90 days)
  Replicas: 3

Consumer: meta-writer-pool
  Type: Pull
  Ack Policy: Explicit
```

---

### 2.3 Search Indexer Worker

**Technology:** Go
**NATS Consumer:** Durable pull consumer on stream `MESSAGES`, consumer name `search-indexer-pool`
**Responsibility:** Index message content into Elasticsearch for full-text search

This consumer reads from the **same stream** as the Message Writer but is a **separate consumer** — NATS JetStream delivers messages independently to each named consumer.

---

### 2.4 Push Notification Worker

**Technology:** Go
**NATS Consumer:** Durable pull consumer on stream `NOTIFICATIONS`
**Responsibility:** Send mobile push notifications (APNs, FCM) for mentions, DMs, and configured alerts

```
Stream: NOTIFICATIONS
  Subjects: notifications.push.>
  Retention: WorkQueue (removed after ack)
  Replicas: 3
```

---

### 2.5 Webhook Dispatcher Worker

**Technology:** Go
**NATS Consumer:** Durable pull consumer on stream `WEBHOOKS`, consumer name `webhook-dispatch-pool`
**Responsibility:** Deliver outgoing webhook HTTP POST requests to registered external bot/integration servers

```
Stream: WEBHOOKS
  Subjects: webhooks.outgoing.>
  Retention: Limits (max age: 7 days)
  Replicas: 3

Consumer: webhook-dispatch-pool
  Type: Pull (shared across N worker instances)
  Ack Policy: Explicit
  Max Ack Pending: 500
  Max Deliver: 5 (with backoff: 5s, 30s, 2min, 15min, 1hr)
  Filter: webhooks.outgoing.>
```

---

## 3. Webhook System Design

The webhook system is the primary integration mechanism for bots, automation, and third-party services.

### 3.1 Webhook Types

**Outgoing Webhooks (Platform → Bot Server):**
When events occur in channels where a webhook is registered, the platform sends an HTTP POST to the bot server's configured URL.

**Incoming Webhooks (Bot Server → Platform):**
A pre-authenticated URL endpoint that allows external services to post messages into specific channels without maintaining a connection.

### 3.2 Outgoing Webhook Payload Format

```json
{
  "webhook_id": "whk_01HZ3K...",
  "event_id": "evt_01HZ3K4M5N6P7Q8R9S0T",
  "event_type": "message.sent",
  "timestamp": "2026-02-01T10:30:00.123Z",
  "channel": {
    "channel_id": "ch_general",
    "name": "general",
    "type": "public"
  },
  "sender": {
    "user_id": "usr_bob",
    "username": "bob",
    "display_name": "Bob Smith"
  },
  "payload": {
    "message_id": "msg_01HZ3K...",
    "text": "Hello, world!",
    "attachments": [],
    "thread_id": null,
    "mentions": ["usr_alice"]
  }
}
```

### 3.3 HTTP Request Headers

```
POST /webhook HTTP/1.1
Host: bot.example.com
Content-Type: application/json
X-Webhook-ID: whk_01HZ3K...
X-Webhook-Signature: sha256=d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592
X-Webhook-Timestamp: 2026-02-01T10:30:00.123Z
X-Event-Type: message.sent
X-Request-ID: req_abc123
User-Agent: CommunicationPlatform-Webhook/1.0
```

### 3.4 Signature Verification (Bot Server Side)

```python
import hmac, hashlib

def verify_signature(payload_body, secret, received_signature):
    expected = "sha256=" + hmac.new(
        secret.encode(), payload_body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, received_signature)
```

### 3.5 Webhook Health & Auto-Disable

The Webhook Dispatcher tracks consecutive failures per webhook. If a webhook reaches a configurable failure threshold (default: 50 consecutive failures), it is automatically disabled.

---

## 4. NATS JetStream Topology

### 4.1 Streams

| Stream Name | Subjects | Retention | Max Age | Replicas | Purpose |
|-------------|----------|-----------|---------|----------|---------|
| `MESSAGES` | `messages.>` | Limits | 30 days | 3 | All message events (send, edit, delete, react, thread_reply) |
| `META` | `channels.>`, `users.>`, `threads.>` | Limits | 90 days | 3 | Channel, user, and thread lifecycle events |
| `NOTIFICATIONS` | `notifications.push.>` | WorkQueue | 7 days | 3 | Push notification delivery queue |
| `WEBHOOKS` | `webhooks.outgoing.>` | Limits | 7 days | 3 | Outgoing webhook delivery queue |
| `RECEIPTS` | `receipts.>` | WorkQueue | 7 days | 3 | Read/delivery receipts for small groups |
| `EPHEMERAL` | `ephemeral.>` | WorkQueue | 5 min | 1 | Ephemeral messages (not persisted, immediate delivery only) |
| `LINK_PREVIEW` | `link_preview.>` | WorkQueue | 1 hour | 3 | Link preview fetch requests and results |
| `FILES` | `files.>` | WorkQueue | 24 hours | 3 | File upload processing (virus scan, thumbnails, metadata) |
| `SYSTEM` | `system.>` | Limits | 7 days | 3 | Audit logs, admin events, internal signals |
| `AUTH` | `auth.>` | Limits | 7 days | 3 | User login/logout events, session events |
| `BOTS` | `bots.>` | Limits | 7 days | 3 | Bot events (slash commands, interactive actions) |

### 4.2 Consumers

| Consumer Name | Stream | Type | Filter | Purpose |
|---------------|--------|------|--------|---------|
| `msg-writer-pool` | MESSAGES | Pull, Durable | `messages.send.>`, `messages.edit.>`, `messages.delete.>`, `messages.thread_reply.>` | Write messages to Cassandra |
| `fan-out-pool` | MESSAGES | Pull, Durable | `messages.>` | Fan-out channel events to instance inboxes |
| `search-indexer-pool` | MESSAGES | Pull, Durable | `messages.send.>`, `messages.edit.>` | Index to Elasticsearch |
| `fan-out-meta` | META | Pull, Durable | `channels.member.>` | Update routing table on membership changes |
| `meta-writer-pool` | META | Pull, Durable | `channels.>`, `users.>` | Write metadata to MongoDB |
| `push-worker-pool` | NOTIFICATIONS | Pull, Durable | `notifications.push.>` | Deliver push notifications |
| `webhook-dispatch-pool` | WEBHOOKS | Pull, Durable | `webhooks.outgoing.>` | Deliver outgoing webhooks |
| `receipt-writer-pool` | RECEIPTS | Pull, Durable | `receipts.>` | Persist per-message receipts (small groups) |
| `ephemeral-fan-out` | EPHEMERAL | Pull, Durable | `ephemeral.>` | Fan-out ephemeral messages to target user only |
| `link-preview-pool` | LINK_PREVIEW | Pull, Durable | `link_preview.fetch.>` | Fetch URL metadata and publish previews |
| `file-virus-scan` | FILES | Pull, Durable | `files.uploaded` | Virus scan uploaded files |
| `file-thumbnail` | FILES | Pull, Durable | `files.uploaded` | Generate thumbnails for images/videos |
| `file-metadata` | FILES | Pull, Durable | `files.uploaded` | Extract file metadata (dimensions, duration, etc.) |
| `auth-audit-pool` | AUTH | Pull, Durable | `auth.>` | Audit logging for authentication events |

### 4.3 Core NATS Subjects (Ephemeral, No Persistence)

| Subject | Publisher | Subscriber | Purpose |
|---------|-----------|------------|---------|
| `instance.events.{instance_id}` | Fan-Out Service | Notification Service (one per instance) | Deliver pre-routed channel events |
| `client.typing.{channel_id}` | Notification Service | Fan-Out Service | Forward typing indicators |
| `presence.change` | Notification Service | Fan-Out Service | Broadcast online/offline transitions |
| `session.logout` | Auth Service | Notification Service | Terminate WebSocket for logged-out session |

---

## 5. Redis/Valkey Usage

Redis (or Valkey, the open-source fork) provides ephemeral state storage with TTL support and pub/sub for real-time change notifications.

| Key Pattern | Type | Value | TTL | Purpose |
|-------------|------|-------|-----|---------|
| `presence:user:{user_id}` | Hash | `{status:"online", instance_id:"notif-07"}` | 120s | User online/offline status |
| `typing:{channel_id}:{user_id}` | String | `1` | 5s | Typing indicator (auto-expires) |
| `permissions:{channel_id}` | Hash | `{roles:..., members_count:N}` | 300s | Cached permission data |
| `ratelimit:user:{user_id}:msg` | String | count | 60s | Per-user rate limiting (INCR + EXPIRE) |
| `ratelimit:webhook:{webhook_id}` | String | count | 60s | Per-webhook rate limiting |
| `instance:{instance_id}` | Hash | `{address:"10.0.1.7", connections:8421}` | 30s | Notification Service discovery |
| `webhook:channels:{channel_id}` | Set | webhook_ids | 300s | Channels with active webhooks |
| `read-pointer:{user_id}:{channel_id}` | Hash | `{last_read_id:..., last_read_at:...}` | none | Per-user per-channel read position |
| `channel-latest:{channel_id}` | Hash | `{latest_id:..., latest_at:..., message_seq:N}` | none | Most recent message per channel |
| `thread-read:{user_id}:{thread_id}` | Hash | `{last_read_id:..., last_read_at:...}` | 30 days | Per-user per-thread read position |
| `thread-latest:{thread_id}` | Hash | `{latest_id:..., latest_at:..., reply_count:N}` | none | Most recent reply per thread |
| `client-state:{user_id}` | Hash | `{last_event_seq:N, instance_id:..., active_channels:[...]}` | 24 hours | Client reconnection state |
| `auth_state:{state}` | Hash | `{state, code_verifier, redirect_uri, client_type}` | 5 min | OIDC auth state (PKCE) |
| `session:{session_id}` | Hash | `{user_id, client_type, device_id, ip, created_at, last_active_at}` | 7 days | User session |
| `refresh_token:{session_id}` | Hash | `{token (encrypted), user_id, client_type, created_at, last_used_at}` | 7 days | Encrypted refresh token |
| `user_sessions:{user_id}` | Set | session_ids | none | Active sessions per user |
| `ws_token:{token}` | Hash | `{user_id, session_id, created_at}` | 30 sec | One-time WebSocket auth token |
| `ratelimit:bot:{bot_id}:messages` | String | count | 60 sec | Bot message rate limit |
| `ratelimit:bot:{bot_id}:api` | String | count | 60 sec | Bot API rate limit |
| `bot:response_url:{token}` | Hash | `{channel_id, message_id, bot_id}` | 30 min | Slash command response URL context |
| `push_prefs:{user_id}` | Hash | `{push_enabled, dm_message, mention, quiet_start, quiet_end}` | 1 hour | Push notification preferences cache |
| `push_recent:{user_id}:{channel_id}` | String | `1` | 30 sec | Push deduplication window |
| `unread:dm:{user_id}` | String | count | none | Unread DM count for badge |
| `unread:mentions:{user_id}` | String | count | none | Unread mentions count for badge |
| `badge:{user_id}` | String | count | none | Total badge count cache |

### Pub/Sub Channels

| Channel | Publisher | Subscriber | Purpose |
|---------|-----------|------------|---------|
| `presence:changes` | Notification Service | Fan-Out Service | User online/offline notifications |
| `typing:changes:{channel_id}` | Notification Service | Fan-Out Service | Typing indicator broadcasts |

---

## 6. Subscription & Fan-Out Architecture

This section describes the core architectural innovation that enables support for users belonging to hundreds of thousands of channels.

### 6.1 The Problem: Subscription Explosion

With the naive per-channel subscription model:

| Parameter | Value |
|-----------|-------|
| Concurrent online users | 100,000 |
| Average channels per user | 100,000 |
| Notification Service instances | 10 |

**Naive model:** 10K users × 100K channels = **1 billion NATS subscriptions per instance**. This is physically impossible.

### 6.2 The Solution: Three-Level Fan-Out Model

```
┌─────────────────────────────────────────────────────────────────────┐
│              Three-Level Fan-Out Architecture                        │
│                                                                      │
│  Level 1: Event Ingestion                                            │
│  ┌──────────────┐     ┌─────────────────────────┐                    │
│  │ Command Svc  │────►│  NATS JetStream          │                   │
│  │              │     │  MESSAGES stream          │                   │
│  └──────────────┘     │  messages.send.ch_general │                   │
│                       └────────────┬────────────┘                    │
│                                    │                                 │
│  Level 2: Instance-Level Routing   │  Pull Consumer                  │
│                       ┌────────────▼────────────┐                    │
│                       │  Fan-Out Service         │                    │
│                       │                          │                    │
│                       │  Routing Table:          │                    │
│                       │  ch_general → {          │                    │
│                       │    notif-01: [alice,bob] │                    │
│                       │    notif-02: [carol,dan] │                    │
│                       │    notif-07: [eve,frank] │                    │
│                       │  }                       │                    │
│                       └───┬───────┬──────┬──────┘                    │
│                           │       │      │                           │
│                   ┌───────┘       │      └────────┐                  │
│                   ▼               ▼               ▼                  │
│            Core NATS:      Core NATS:       Core NATS:               │
│    instance.events.   instance.events.  instance.events.             │
│         notif-01          notif-02          notif-07                 │
│                                                                      │
│  Level 3: Local WebSocket Delivery                                   │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                  │
│  │ Notif Svc 01 │ │ Notif Svc 02 │ │ Notif Svc 07 │                  │
│  │              │ │              │ │              │                  │
│  │ Local Table: │ │ Local Table: │ │ Local Table: │                  │
│  │ ch_general → │ │ ch_general → │ │ ch_general → │                  │
│  │  [ws_alice,  │ │  [ws_carol,  │ │  [ws_eve,    │                  │
│  │   ws_bob]    │ │   ws_dan]    │ │   ws_frank]  │                  │
│  └───┬────┬─────┘ └───┬────┬─────┘ └───┬────┬─────┘                  │
│      │    │            │    │          │    │                        │
│      ▼    ▼            ▼    ▼          ▼    ▼                        │
│   Alice  Bob        Carol  Dan       Eve  Frank                      │
│   (WS)  (WS)       (WS)  (WS)      (WS) (WS)                         │
└─────────────────────────────────────────────────────────────────────┘
```

**Level 1 — Event Ingestion:** The Command Service publishes a single event to NATS JetStream. One event, one subject, one write.

**Level 2 — Instance-Level Routing:** The Fan-Out Service consumes the event and consults its routing table: "Which instances have online users in this channel?" It publishes to each relevant instance's inbox subject.

**Level 3 — Local WebSocket Delivery:** Each Notification Service instance receives events on its single inbox subscription and routes locally to WebSocket connections.

### 6.3 Routing Table Design

The Fan-Out Service maintains two in-memory data structures:

**Primary Index — Channel → Instance Mapping:**

```go
type ChannelRouting struct {
    mu        sync.RWMutex
    channels  map[string]map[string]*InstanceEntry  // channel_id → instance_id → entry
}

type InstanceEntry struct {
    InstanceID string
    UserIDs    map[string]struct{}  // online users on this instance in this channel
}
```

**Secondary Index — User → Channels Mapping:**

```go
type UserIndex struct {
    mu    sync.RWMutex
    users map[string]*UserEntry  // user_id → entry
}

type UserEntry struct {
    InstanceID string
    ChannelIDs map[string]struct{}
}
```

**Memory Analysis:**

| Data | Per-Entry Size | Count | Total Memory |
|------|---------------|-------|--------------|
| Channel → Instance mapping | ~80 bytes | 500K channels × 5 instances | ~200 MB |
| Instance → Users per channel | ~40 bytes | 100K users × 200 active channels | ~800 MB |
| User → Channels index | ~40 bytes | 100K users × 200 active channels | ~800 MB |
| **Total per Fan-Out Worker** | | | **~1.8 GB** |

### 6.4 Scaling Analysis

**Fan-Out Service throughput at 50,000 messages/sec:**

- 1 pull from JetStream (batched: Fetch 200 at a time)
- Average 5 Core NATS publishes per message
- Total outbound: 250,000 Core NATS publishes/sec
- 2 Fan-Out Workers handle this with room for spikes

**Worst case: 1M-member channel, all online across 100 instances:**

- 1 message → 100 Core NATS publishes (one per instance)
- Each instance does local fan-out to ~10,000 WebSocket connections
- **Savings: 10,000x reduction in NATS message volume**

**NATS subscription count (system-wide):**

| Component | NATS Subscriptions |
|-----------|-------------------|
| Notification Service (100 instances) | 100 |
| Fan-Out Service (3 workers) | 6 |
| Message Writers (5 workers) | 5 |
| Other workers | ~10 |
| **System total** | **~121** |

---

## 7. Event Replay and State Reconstruction

NATS JetStream retains message events for 30 days, enabling:

### 7.1 Cold Start / New Consumer Replay

When a new Message Writer instance joins, it binds to the durable consumer `msg-writer-pool`. Unacknowledged messages are redelivered automatically.

### 7.2 Database Reconstruction

If Cassandra data is lost, a recovery consumer can replay all retained events:

```
Consumer: recovery-replay
  Stream: MESSAGES
  Deliver Policy: DeliverAll (or DeliverByStartTime)
  Type: Pull
  Replay Policy: Instant
```

### 7.3 Fan-Out Service Recovery

The routing table is entirely ephemeral and reconstructable from Redis (presence) + MongoDB (memberships). No persistent state needs to be backed up.

---

## 8. Technical Considerations

### 8.1 Scalability

| Component | Scaling Mechanism | Bottleneck Mitigation |
|-----------|-------------------|----------------------|
| API Gateway | Horizontal (stateless) | Auto-scaling based on connections |
| Command Service | Horizontal (stateless) | NATS publish is non-blocking |
| Fan-Out Service | Horizontal (pull consumers) | NATS distributes messages automatically |
| Message Writers | Horizontal (pull consumers) | Add instances as needed |
| Query Service | Horizontal (stateless) | Cassandra token-aware routing |
| Notification Service | Horizontal (per instance) | One NATS subscription per instance |
| NATS JetStream | Clustered (R=3) | Super-clusters for geo-distribution |
| Cassandra | Horizontal (vnodes) | Partition design prevents hotspots |

### 8.2 Reliability and Failure Modes

| Failure Scenario | Behavior | Recovery |
|-----------------|----------|----------|
| Command Service crash | Gateway routes to healthy instances | Automatic (no state) |
| Message Writer crash | Unacked messages redelivered | Automatic redelivery |
| Fan-Out Service crash | Events buffer in JetStream | Rebuilds table in 30–60s |
| Notification Service crash | Clients reconnect elsewhere | Fan-Out routing updated |
| NATS node failure (1 of 3) | RAFT quorum maintained | Replace and re-replicate |
| Cassandra node failure | Quorum reads/writes continue | Auto-repair via streaming |

### 8.3 Consistency Model

- **Write ordering:** NATS JetStream assigns monotonically increasing sequence numbers
- **Read-after-write gap:** Brief window (~50ms) where GET may not include new messages
- **Exactly-once processing:** Via publisher deduplication + idempotent database writes
- **Routing table consistency:** Eventually consistent with 1–5ms lag

### 8.4 Security

- **Transport:** TLS everywhere
- **Authentication:** JWT tokens validated at gateway; NATS uses NKey authentication
- **Authorization:** Channel-level permissions checked in Command Service
- **Webhooks:** HMAC-SHA256 signing; URL allowlist/denylist for SSRF prevention
- **Data at rest:** Cassandra and MongoDB encryption; NATS JetStream encryption

### 8.5 Observability

- **Distributed Tracing:** OpenTelemetry traces with `trace_id` and `correlation_id`
- **Metrics (Prometheus):** NATS lag, request rates, processing latency, connection counts
- **Logging:** Structured JSON logs with correlation IDs
- **Alerting:** Consumer lag, NACK rate, WebSocket drop rate, GC pauses

---

## 9. Multi-Region Deployment Topology

This section describes the geo-distribution strategy for reducing latency for globally distributed users.

### 9.1 Regional Edge + Global Backbone Architecture

The platform supports multi-region deployment using a **Regional Edge + Global Backbone** pattern:

```
                    ┌─────────────────────────────────────┐
                    │      NATS Super-Cluster (Global)    │
                    │  ┌─────────┐       ┌─────────┐      │
                    │  │US-East  │◄─────►│US-West  │      │
                    │  │ Cluster │       │ Cluster │      │
                    │  └────┬────┘       └────┬────┘      │
                    │       │   Gateway Links  │          │
                    │  ┌────┴────┐       ┌────┴────┐      │
                    │  │  EU     │◄─────►│  Asia   │      │
                    │  │ Cluster │       │ Cluster │      │
                    │  └─────────┘       └─────────┘      │
                    └─────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐          ┌───────────────┐          ┌───────────────┐
│ Regional Edge │          │ Regional Edge │          │ Regional Edge │
│   US-East     │          │      EU       │          │     Asia      │
│   (Primary)   │          │               │          │               │
│               │          │               │          │               │
│ • Fan-Out Svc │          │ • Notif Svc   │          │ • Notif Svc   │
│ • Notif Svc   │          │ • Query Svc   │          │ • Query Svc   │
│ • Query Svc   │          │ • Redis Read  │          │ • Redis Read  │
│ • Cmd Svc     │          │ • Cassandra   │          │ • Cassandra   │
│ • Workers     │          │   (LOCAL_ONE) │          │   (LOCAL_ONE) │
│ • Redis Pri   │          │ • Mongo Sec   │          │ • Mongo Sec   │
│ • Cassandra   │          │               │          │               │
│ • Mongo Pri   │          │               │          │               │
└───────────────┘          └───────────────┘          └───────────────┘
```

### 9.2 NATS Super-Cluster with Gateway Links

Each region runs its own 3-node NATS cluster. Gateway links connect regional clusters with **interest-based forwarding**: only events with subscribers in remote regions traverse gateways.

**Key benefit:** A message in a US-only channel never crosses the Atlantic — there's no subscriber interest in EU.

| Stream | Replication | Cross-Region Behavior |
|--------|-------------|----------------------|
| MESSAGES | R=3 within primary DC | Events fan out via gateways to regional instance inboxes |
| META | R=3 within primary DC | Consumed by workers in primary region only |
| Core NATS subjects | No persistence | Interest-based gateway forwarding |

### 9.3 Global Fan-Out Service

At 50K users, the routing table is ~2GB — well within single-instance memory. The Fan-Out Service runs in the primary region (US-East) and publishes to all Notification Service instances globally via NATS gateway links.

**Rationale for global (not regional) Fan-Out:**
- Single source of truth for routing table
- No distributed consensus complexity
- Cross-region latency penalty (~100-150ms) is acceptable
- Regional Fan-Out would require routing table synchronization

### 9.4 Regional Services

| Service | Deployment | Notes |
|---------|------------|-------|
| **API Gateway** | Every region | GeoDNS routes users to nearest region |
| **Notification Service** | Every region | Users connect to nearest instances via WebSocket |
| **Query Service** | Every region | Reads from regional data store replicas |
| **Command Service** | Primary region only | All writes go to US-East |
| **Fan-Out Service** | Primary region only | Routes events globally via gateways |
| **Workers** | Primary region only | Consume from primary NATS cluster |

### 9.5 Data Store Multi-DC Configuration

**Apache Cassandra:**

```sql
CREATE KEYSPACE chat WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'us-east': 3,
  'eu-west': 3,
  'asia': 3
};
```

| Operation | Consistency | Rationale |
|-----------|-------------|-----------|
| Write | LOCAL_QUORUM | Durability in primary DC |
| Read | LOCAL_ONE | Low latency from regional replica |

**Redis/Valkey:**
- Primary in US-East (all writes)
- Read replicas in EU, Asia (async replication)
- Regional Query Services read from local replica

**MongoDB:**
- Primary-Secondary topology
- Primary in US-East, secondaries in EU/Asia
- Read preference: `secondaryPreferred` with region tags

### 9.6 Latency Targets

| Scenario | Path | Target |
|----------|------|--------|
| Same-region message | User→Regional Notif→User | <50ms |
| Cross-region message | User→Primary→Fan-Out→Gateway→Regional Notif→User | <300ms |
| History query | User→Regional Query→Regional Cassandra | <30ms |
| Presence propagation | User→Primary Redis→Replica | <200ms |

For detailed deployment configuration and failure scenarios, see [Multi-Region Feature Documentation](./features/multi-region.md).

---

## Related Documents

- [Overview](./overview.md) — Executive summary and high-level architecture
- [C4 Diagrams](./diagrams/c4-diagrams.md) — Visual architecture diagrams
- [Sequence Diagrams](./diagrams/sequence-diagrams.md) — Flow diagrams for key operations
- [Features](./features/) — Detailed specifications for threads, reconnection, receipts, etc.
- [Authentication](./features/authentication.md) — OIDC integration and login flows
- [Bot Integration](./features/bot-integration.md) — Bot authentication, webhooks, and API
- [Push Notifications](./features/push-notifications.md) — APNs and FCM push delivery
- [API Reference](./api/api-reference.md) — REST API endpoints and migration guide
- [Security Architecture](./security/security-architecture.md) — Security controls and compliance
- [ADRs](./adrs/) — Architecture decision records
