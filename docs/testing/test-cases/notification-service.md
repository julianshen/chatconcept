# Test Cases: Notification Service

**Service:** Notification Service
**Version:** 1.0
**Last Updated:** 2026-02-02

---

## Overview

The Notification Service manages WebSocket connections and delivers real-time events to clients. Each instance:
- Maintains WebSocket connections for thousands of users
- Subscribes to its own instance inbox (`instance.events.{id}`)
- Routes events to local connections
- Handles presence and typing indicators

---

## Test Categories

1. [WebSocket Connection](#1-websocket-connection)
2. [Authentication](#2-authentication)
3. [Event Delivery](#3-event-delivery)
4. [Local Routing](#4-local-routing)
5. [Presence Management](#5-presence-management)
6. [Multi-Device Handling](#6-multi-device-handling)
7. [Connection Recovery](#7-connection-recovery)
8. [Performance](#8-performance)

---

## 1. WebSocket Connection

### TC-NOTIF-WS-001: Establish WebSocket Connection

**Priority:** Critical
**Type:** Unit, Integration

**Steps:**
1. Client initiates WebSocket upgrade
2. Provide valid authentication token
3. Connection established

**Expected Results:**
- HTTP 101 Switching Protocols
- WebSocket connection open
- Server sends `auth.ok` message

**Go Test:**
```go
func TestWebSocket_ValidAuth_ConnectionEstablished(t *testing.T) {
    server := setupTestServer()
    defer server.Close()

    token := generateValidJWT("usr_alice")
    ws, resp, err := websocket.DefaultDialer.Dial(
        server.URL+"/ws?token="+token,
        nil,
    )
    require.NoError(t, err)
    defer ws.Close()

    assert.Equal(t, http.StatusSwitchingProtocols, resp.StatusCode)

    // Read auth.ok message
    var msg Message
    err = ws.ReadJSON(&msg)
    require.NoError(t, err)
    assert.Equal(t, "auth.ok", msg.Type)
}
```

---

### TC-NOTIF-WS-002: Connection with Invalid Token

**Priority:** Critical
**Type:** Unit

**Expected Results:**
- HTTP 401 Unauthorized
- No WebSocket upgrade
- Error message in response

---

### TC-NOTIF-WS-003: Connection with Expired Token

**Priority:** High
**Type:** Unit

**Expected Results:**
- HTTP 401 Unauthorized
- Error: `token_expired`

---

### TC-NOTIF-WS-004: Connection Limit Per User

**Priority:** Medium
**Type:** Integration

**Preconditions:**
- User already has maximum connections (e.g., 10)

**Steps:**
1. Attempt to open another connection

**Expected Results:**
- Oldest connection closed
- New connection established
- Or: Connection rejected with error

---

### TC-NOTIF-WS-005: Graceful Connection Close

**Priority:** High
**Type:** Unit

**Steps:**
1. Client sends close frame
2. Server acknowledges

**Expected Results:**
- Connection closed cleanly
- User removed from local routing table
- Presence updated to offline (if last connection)

---

### TC-NOTIF-WS-006: Heartbeat/Ping-Pong

**Priority:** High
**Type:** Unit, Integration

**Steps:**
1. Connection established
2. Server sends ping every 30s
3. Client responds with pong

**Expected Results:**
- Connection stays alive
- Pong received within 10s
- Connection closed if no pong

**Go Test:**
```go
func TestWebSocket_Heartbeat_KeepsConnectionAlive(t *testing.T) {
    server := setupTestServer()
    ws := connectWebSocket(server, "usr_alice")
    defer ws.Close()

    // Set pong handler
    pongReceived := make(chan bool)
    ws.SetPongHandler(func(string) error {
        pongReceived <- true
        return nil
    })

    // Wait for ping from server (max 35s)
    go func() {
        for {
            _, _, err := ws.ReadMessage()
            if err != nil {
                return
            }
        }
    }()

    select {
    case <-pongReceived:
        // Success - connection maintained
    case <-time.After(35 * time.Second):
        t.Fatal("no ping received within 35 seconds")
    }
}
```

---

## 2. Authentication

### TC-NOTIF-AUTH-001: One-Time WebSocket Token

**Priority:** High
**Type:** Unit, Integration

**Steps:**
1. Request one-time WS token from API
2. Use token to establish WebSocket
3. Try to reuse same token

**Expected Results:**
- First connection succeeds
- Second connection fails (token consumed)
- Token expires after 30 seconds if unused

---

### TC-NOTIF-AUTH-002: Session Invalidation

**Priority:** High
**Type:** Integration

**Steps:**
1. User logged in with active WebSocket
2. User logs out from another device
3. Auth Service publishes `session.logout`

**Expected Results:**
- WebSocket receives `session.terminated` message
- Connection closed by server
- Client redirected to login

---

### TC-NOTIF-AUTH-003: Token Refresh Mid-Session

**Priority:** Medium
**Type:** Integration

**Steps:**
1. WebSocket connected
2. JWT approaches expiry
3. Client sends refresh request

**Expected Results:**
- New JWT issued
- Connection continues
- No interruption in event delivery

---

## 3. Event Delivery

### TC-NOTIF-EVT-001: Deliver Message Event

**Priority:** Critical
**Type:** Unit, Integration

**Preconditions:**
- Alice connected to this instance
- Alice is member of ch_general

**Steps:**
1. Fan-Out publishes to instance inbox
2. Instance routes to Alice's WebSocket

**Expected Results:**
- Alice receives message event
- Event contains full message payload
- Delivery latency < 50ms

**Go Test:**
```go
func TestNotificationService_DeliverMessageEvent(t *testing.T) {
    ns := setupNotificationService()
    aliceWS := connectUser(ns, "usr_alice")
    defer aliceWS.Close()

    // Simulate fan-out publishing to instance inbox
    event := Event{
        Type:      "message.sent",
        ChannelID: "ch_general",
        Payload: map[string]interface{}{
            "message_id": "msg_123",
            "text":       "Hello!",
            "sender_id":  "usr_bob",
        },
    }
    ns.localRouter.Route(event, []string{"usr_alice"})

    // Verify Alice receives
    var received Event
    err := aliceWS.ReadJSON(&received)
    require.NoError(t, err)
    assert.Equal(t, "message.sent", received.Type)
}
```

---

### TC-NOTIF-EVT-002: Deliver Typing Indicator

**Priority:** High
**Type:** Unit

**Expected Results:**
- Typing indicator delivered to channel members
- Excludes the user who is typing
- Low latency (ephemeral)

---

### TC-NOTIF-EVT-003: Deliver Read Receipt

**Priority:** High
**Type:** Unit

**Expected Results:**
- Read receipt delivered to message sender
- For DM/small groups only
- Contains reader info and timestamp

---

### TC-NOTIF-EVT-004: Event Ordering

**Priority:** High
**Type:** Unit, Integration

**Steps:**
1. Send 100 messages rapidly
2. Verify delivery order

**Expected Results:**
- Messages delivered in send order
- No out-of-order delivery
- Sequence numbers maintained

---

### TC-NOTIF-EVT-005: Delivery to Offline User

**Priority:** Medium
**Type:** Unit

**Preconditions:**
- User was online, just disconnected
- Event arrives for user

**Expected Results:**
- Event dropped (no queue)
- No error
- User catches up on reconnect via REST

---

## 4. Local Routing

### TC-NOTIF-LR-001: Route Event to Single User

**Priority:** Critical
**Type:** Unit

**Expected Results:**
- Event delivered to exactly one user
- O(1) routing lookup

---

### TC-NOTIF-LR-002: Route Event to Channel Members

**Priority:** Critical
**Type:** Unit

**Preconditions:**
- 10 users from ch_general connected to this instance

**Steps:**
1. Channel event arrives

**Expected Results:**
- Event delivered to all 10 users
- Single iteration over local table
- No delivery to non-members

**Go Test:**
```go
func TestLocalRouter_ChannelEvent_DeliveredToAllMembers(t *testing.T) {
    router := NewLocalRouter()

    // Connect 10 users
    connections := make([]*MockWebSocket, 10)
    for i := 0; i < 10; i++ {
        userID := fmt.Sprintf("usr_%d", i)
        ws := &MockWebSocket{}
        router.AddConnection(userID, "ch_general", ws)
        connections[i] = ws
    }

    // Route channel event
    event := Event{Type: "message.sent", ChannelID: "ch_general"}
    router.RouteToChannel("ch_general", event)

    // Verify all received
    for i, ws := range connections {
        assert.Len(t, ws.Sent, 1, "user %d did not receive event", i)
    }
}
```

---

### TC-NOTIF-LR-003: Route DM Event

**Priority:** High
**Type:** Unit

**Preconditions:**
- DM between Alice and Bob
- Only Alice on this instance

**Expected Results:**
- Only Alice receives event
- Bob's delivery handled by his instance

---

### TC-NOTIF-LR-004: Local Table Update on Join

**Priority:** High
**Type:** Unit

**Steps:**
1. User joins new channel
2. Membership event received

**Expected Results:**
- User added to channel's local connection list
- Future channel events delivered to user

---

### TC-NOTIF-LR-005: Local Table Update on Leave

**Priority:** High
**Type:** Unit

**Steps:**
1. User leaves channel
2. Membership event received

**Expected Results:**
- User removed from channel's local list
- No more events for that channel

---

## 5. Presence Management

### TC-NOTIF-PRES-001: User Comes Online

**Priority:** Critical
**Type:** Unit, Integration

**Steps:**
1. WebSocket connection established
2. User added to local routing

**Expected Results:**
- Redis presence updated: `online`
- `presence.changed` published
- Fan-Out updates routing table

**Go Test:**
```go
func TestPresence_UserConnects_SetsOnline(t *testing.T) {
    ns := setupNotificationService()
    redis := ns.redis.(*MockRedis)

    connectUser(ns, "usr_alice")

    // Verify Redis updated
    presence, err := redis.HGetAll(context.Background(), "presence:user:usr_alice").Result()
    require.NoError(t, err)
    assert.Equal(t, "online", presence["status"])
    assert.Equal(t, ns.instanceID, presence["instance_id"])
}
```

---

### TC-NOTIF-PRES-002: User Goes Offline

**Priority:** Critical
**Type:** Unit, Integration

**Steps:**
1. WebSocket connection closed
2. Last connection for user

**Expected Results:**
- Redis presence updated: `offline`
- `presence.changed` published
- `last_seen` timestamp set

---

### TC-NOTIF-PRES-003: Presence TTL Refresh

**Priority:** High
**Type:** Unit

**Steps:**
1. User connected
2. Heartbeat received

**Expected Results:**
- Redis key TTL refreshed to 120s
- Prevents false offline detection

---

### TC-NOTIF-PRES-004: Multiple Devices Same User

**Priority:** High
**Type:** Unit

**Preconditions:**
- Alice connected from 2 devices

**Steps:**
1. Device 1 disconnects

**Expected Results:**
- User still shows online (device 2 active)
- Presence not changed to offline

---

### TC-NOTIF-PRES-005: Client State Tracking

**Priority:** High
**Type:** Unit

**Steps:**
1. Client sends heartbeat with state: `active`
2. No interaction for 60s
3. Client sends heartbeat with state: `idle`

**Expected Results:**
- Redis `client_state` updated
- Used for push notification decisions

---

## 6. Multi-Device Handling

### TC-NOTIF-MD-001: Sync Read State Across Devices

**Priority:** High
**Type:** Unit, Integration

**Steps:**
1. Alice on Desktop (this instance) and Mobile (other instance)
2. Alice reads on Mobile
3. `user.sync.alice` event published

**Expected Results:**
- Desktop receives `sync.read` message
- Unread badge updates
- No duplicate notification

---

### TC-NOTIF-MD-002: Receive Event on All Devices

**Priority:** High
**Type:** Unit

**Preconditions:**
- Alice connected from 3 devices to this instance

**Steps:**
1. Message event for Alice arrives

**Expected Results:**
- All 3 WebSocket connections receive event
- Identical payload to all

---

### TC-NOTIF-MD-003: Typing From One Device

**Priority:** Medium
**Type:** Unit

**Steps:**
1. Alice typing on Desktop
2. Alice's Mobile also connected

**Expected Results:**
- Mobile does NOT show self-typing indicator
- Other users see Alice typing

---

## 7. Connection Recovery

### TC-NOTIF-REC-001: Client Reconnect with Last Event Seq

**Priority:** High
**Type:** Integration

**Steps:**
1. Client disconnects
2. Reconnects with `last_event_seq: 12345`
3. Gap < 2 minutes

**Expected Results:**
- JetStream replay from seq 12345
- Missed events delivered
- No duplicates

**Go Test:**
```go
func TestReconnect_ShortGap_JetStreamReplay(t *testing.T) {
    ns := setupNotificationService()
    ws := connectUserWithState(ns, "usr_alice", ReconnectState{
        LastEventSeq: 12345,
        LastSeen:     time.Now().Add(-1 * time.Minute),
    })
    defer ws.Close()

    // Verify catchup message received
    var msg Message
    err := ws.ReadJSON(&msg)
    require.NoError(t, err)
    assert.Equal(t, "catchup.start", msg.Type)

    // Read catchup events...
    for {
        err = ws.ReadJSON(&msg)
        if err != nil || msg.Type == "catchup.end" {
            break
        }
    }

    assert.Equal(t, "catchup.end", msg.Type)
}
```

---

### TC-NOTIF-REC-002: Client Reconnect After Long Offline

**Priority:** High
**Type:** Integration

**Steps:**
1. Client offline for > 24 hours
2. Reconnects

**Expected Results:**
- `full_refresh_required` message sent
- Client fetches state via REST
- No JetStream replay (too old)

---

### TC-NOTIF-REC-003: Instance Failover

**Priority:** High
**Type:** Integration

**Steps:**
1. Instance crashes
2. Clients reconnect to different instances

**Expected Results:**
- Presence updated to new instances
- Fan-Out routing table updated
- Messages continue flowing

---

## 8. Performance

### TC-NOTIF-PERF-001: Connection Capacity

**Priority:** Critical
**Type:** Performance

**Test:**
- Open 10,000 WebSocket connections to single instance
- Send heartbeats from all

**Expected Results:**
- All connections maintained
- Memory usage < 2GB
- Heartbeat processing < 100ms total

---

### TC-NOTIF-PERF-002: Event Delivery Throughput

**Priority:** Critical
**Type:** Performance

**Test:**
- 10,000 connected users
- 5,000 events/sec to instance inbox
- Events fan out to ~50 users each

**Expected Results:**
- 250,000 WebSocket writes/sec
- No message drops
- Latency p99 < 50ms

---

### TC-NOTIF-PERF-003: Memory Per Connection

**Priority:** High
**Type:** Performance

**Benchmark:**
```go
func BenchmarkMemoryPerConnection(b *testing.B) {
    ns := setupNotificationService()

    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    before := m.HeapAlloc

    for i := 0; i < b.N; i++ {
        connectUser(ns, fmt.Sprintf("usr_%d", i))
    }

    runtime.ReadMemStats(&m)
    after := m.HeapAlloc

    perConnection := (after - before) / uint64(b.N)
    b.ReportMetric(float64(perConnection), "bytes/conn")
}
```

**Expected Results:**
- < 10KB per connection
- Stable under sustained load

---

### TC-NOTIF-PERF-004: Broadcast Latency

**Priority:** High
**Type:** Performance

**Test:**
- Channel with 1000 users on this instance
- Measure time to write to all WebSockets

**Expected Results:**
- Broadcast complete in < 10ms
- Parallel writes used

---

## Test Execution Summary

| Category | Total | Critical | High | Medium | Low |
|----------|-------|----------|------|--------|-----|
| WebSocket Connection | 6 | 2 | 3 | 1 | 0 |
| Authentication | 3 | 0 | 2 | 1 | 0 |
| Event Delivery | 5 | 1 | 3 | 1 | 0 |
| Local Routing | 5 | 2 | 3 | 0 | 0 |
| Presence Management | 5 | 2 | 3 | 0 | 0 |
| Multi-Device Handling | 3 | 0 | 2 | 1 | 0 |
| Connection Recovery | 3 | 0 | 3 | 0 | 0 |
| Performance | 4 | 2 | 2 | 0 | 0 |
| **Total** | **34** | **9** | **21** | **4** | **0** |
