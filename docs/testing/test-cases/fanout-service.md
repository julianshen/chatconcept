# Test Cases: Fan-Out Service

**Service:** Fan-Out Service
**Version:** 1.0
**Last Updated:** 2026-02-02

---

## Overview

The Fan-Out Service is the critical routing component that:
- Consumes events from NATS JetStream
- Maintains an in-memory routing table (channel → instance mapping)
- Routes events to Notification Service instance inboxes
- Handles presence changes and membership updates

**Critical Invariant:** Routes to **instances**, not individual users. This enables O(instances) fan-out instead of O(users).

---

## Test Categories

1. [Routing Table Operations](#1-routing-table-operations)
2. [Event Processing](#2-event-processing)
3. [Presence Handling](#3-presence-handling)
4. [Membership Changes](#4-membership-changes)
5. [Concurrency](#5-concurrency)
6. [Recovery](#6-recovery)
7. [Performance](#7-performance)

---

## 1. Routing Table Operations

### TC-FAN-RT-001: Add User to Routing Table

**Priority:** Critical
**Type:** Unit

**Steps:**
1. User `alice` comes online on instance `notif-07`
2. Alice is member of channels: `ch_general`, `ch_dev`

**Expected Results:**
- Channel index updated: `ch_general → {notif-07: [alice]}`
- Channel index updated: `ch_dev → {notif-07: [alice]}`
- User index updated: `alice → {instance: notif-07, channels: [ch_general, ch_dev]}`

**Go Test:**
```go
func TestRoutingTable_AddUser(t *testing.T) {
    rt := NewRoutingTable()
    membershipCache := &MockMembershipCache{
        Memberships: map[string][]string{
            "usr_alice": {"ch_general", "ch_dev"},
        },
    }

    rt.AddUser("usr_alice", "notif-07", membershipCache)

    // Verify channel index
    entry := rt.GetChannel("ch_general")
    require.NotNil(t, entry)
    assert.Contains(t, entry.Instances["notif-07"].Users, "usr_alice")

    // Verify user index
    userEntry := rt.GetUser("usr_alice")
    require.NotNil(t, userEntry)
    assert.Equal(t, "notif-07", userEntry.InstanceID)
    assert.Contains(t, userEntry.Channels, "ch_general")
}
```

---

### TC-FAN-RT-002: Remove User from Routing Table

**Priority:** Critical
**Type:** Unit

**Preconditions:**
- Alice is in routing table on notif-07
- Alice is in channels: ch_general, ch_dev
- Bob is also in ch_general on notif-07

**Steps:**
1. Alice goes offline

**Expected Results:**
- Alice removed from all channel entries
- Alice removed from user index
- `ch_general → notif-07` still exists (Bob is there)
- `ch_dev → notif-07` removed (no users left on this instance)

**Go Test:**
```go
func TestRoutingTable_RemoveUser_CleansEmptyInstances(t *testing.T) {
    rt := NewRoutingTable()
    // Setup: Alice and Bob in ch_general, only Alice in ch_dev
    rt.AddUser("usr_alice", "notif-07", &MockMembershipCache{
        Memberships: map[string][]string{
            "usr_alice": {"ch_general", "ch_dev"},
        },
    })
    rt.AddUser("usr_bob", "notif-07", &MockMembershipCache{
        Memberships: map[string][]string{
            "usr_bob": {"ch_general"},
        },
    })

    // Act: Remove Alice
    rt.RemoveUser("usr_alice")

    // Assert
    assert.Nil(t, rt.GetUser("usr_alice"))

    // ch_general still has notif-07 (Bob)
    generalEntry := rt.GetChannel("ch_general")
    assert.Contains(t, generalEntry.Instances, "notif-07")

    // ch_dev has no instances (was only Alice)
    devEntry := rt.GetChannel("ch_dev")
    assert.Empty(t, devEntry.Instances)
}
```

---

### TC-FAN-RT-003: Lookup Channel Routing

**Priority:** Critical
**Type:** Unit

**Preconditions:**
- ch_general has users on notif-01, notif-07, notif-12

**Steps:**
1. Lookup routing for ch_general

**Expected Results:**
- Returns: `[notif-01, notif-07, notif-12]`
- O(1) lookup time

---

### TC-FAN-RT-004: Lookup Empty Channel

**Priority:** High
**Type:** Unit

**Preconditions:**
- Channel exists but no online users

**Expected Results:**
- Returns empty instance list
- No error

---

### TC-FAN-RT-005: Lookup Nonexistent Channel

**Priority:** High
**Type:** Unit

**Expected Results:**
- Returns empty instance list
- No error (channel might have no online members)

---

## 2. Event Processing

### TC-FAN-EVT-001: Route Message to Single Instance

**Priority:** Critical
**Type:** Unit, Integration

**Preconditions:**
- ch_small has users only on notif-07

**Steps:**
1. Message event arrives for ch_small

**Expected Results:**
- Single publish to `instance.events.notif-07`
- Event includes full message payload

**Go Test:**
```go
func TestFanOut_SingleInstance_OnePublish(t *testing.T) {
    rt := NewRoutingTable()
    rt.SetChannelInstances("ch_small", []string{"notif-07"})

    publisher := &MockNATSPublisher{}
    fanout := NewFanOutService(rt, publisher)

    event := MessageEvent{
        ChannelID: "ch_small",
        MessageID: "msg_123",
        Text:      "Hello",
    }

    fanout.ProcessEvent(event)

    require.Len(t, publisher.Published, 1)
    assert.Equal(t, "instance.events.notif-07", publisher.Published[0].Subject)
}
```

---

### TC-FAN-EVT-002: Route Message to Multiple Instances

**Priority:** Critical
**Type:** Unit, Integration

**Preconditions:**
- ch_general has users on notif-01, notif-07, notif-12

**Steps:**
1. Message event arrives for ch_general

**Expected Results:**
- Three publishes: notif-01, notif-07, notif-12
- All receive identical event payload
- Single serialization (serialize once, publish N times)

---

### TC-FAN-EVT-003: Route Message to Large Channel (100 instances)

**Priority:** High
**Type:** Unit, Performance

**Preconditions:**
- ch_company has users on 100 different instances

**Expected Results:**
- 100 publishes completed
- Batch publishing used for efficiency
- Total time < 10ms

---

### TC-FAN-EVT-004: Route Typing Indicator (Exclude Origin)

**Priority:** High
**Type:** Unit

**Preconditions:**
- ch_general has users on notif-01, notif-07, notif-12
- Typing event originated from notif-07

**Steps:**
1. Typing event arrives from notif-07

**Expected Results:**
- Publishes to notif-01 and notif-12 only
- Does NOT publish back to notif-07 (origin)

**Go Test:**
```go
func TestFanOut_TypingIndicator_ExcludesOrigin(t *testing.T) {
    rt := NewRoutingTable()
    rt.SetChannelInstances("ch_general", []string{"notif-01", "notif-07", "notif-12"})

    publisher := &MockNATSPublisher{}
    fanout := NewFanOutService(rt, publisher)

    event := TypingEvent{
        ChannelID:      "ch_general",
        UserID:         "usr_alice",
        OriginInstance: "notif-07",
        Typing:         true,
    }

    fanout.ProcessTypingEvent(event)

    subjects := extractSubjects(publisher.Published)
    assert.Contains(t, subjects, "instance.events.notif-01")
    assert.Contains(t, subjects, "instance.events.notif-12")
    assert.NotContains(t, subjects, "instance.events.notif-07")
}
```

---

### TC-FAN-EVT-005: Batch Event Processing

**Priority:** High
**Type:** Unit, Integration

**Steps:**
1. Pull batch of 200 events from JetStream
2. Process all events

**Expected Results:**
- All events processed
- Batch acknowledged after successful processing
- Publishes batched where possible

---

### TC-FAN-EVT-006: Event Processing Failure

**Priority:** High
**Type:** Unit

**Preconditions:**
- NATS Core publish fails

**Steps:**
1. Process event
2. Publish to instance inbox fails

**Expected Results:**
- Event NAKed (negative acknowledgment)
- Event redelivered after delay
- Error logged with context

---

## 3. Presence Handling

### TC-FAN-PRES-001: User Comes Online

**Priority:** Critical
**Type:** Unit, Integration

**Steps:**
1. Redis pub/sub receives: `alice:online:notif-07`
2. Fetch Alice's memberships (100K channels)
3. Update routing table

**Expected Results:**
- All 100K channels updated in routing table
- Processing time < 50ms
- Memory usage bounded

**Go Test:**
```go
func TestPresenceHandler_UserOnline_UpdatesRoutingTable(t *testing.T) {
    rt := NewRoutingTable()
    membershipCache := &MockMembershipCache{
        Memberships: map[string][]string{
            "usr_alice": generateChannelIDs(1000), // 1K channels for test
        },
    }

    handler := NewPresenceHandler(rt, membershipCache)

    start := time.Now()
    handler.HandlePresenceChange(PresenceEvent{
        UserID:     "usr_alice",
        InstanceID: "notif-07",
        Status:     "online",
    })
    duration := time.Since(start)

    assert.Less(t, duration, 50*time.Millisecond)
    assert.NotNil(t, rt.GetUser("usr_alice"))
}
```

---

### TC-FAN-PRES-002: User Goes Offline

**Priority:** Critical
**Type:** Unit, Integration

**Steps:**
1. Redis pub/sub receives: `alice:offline`
2. Remove Alice from routing table

**Expected Results:**
- Alice removed from all channel entries
- Empty instance entries cleaned up
- Processing time < 50ms

---

### TC-FAN-PRES-003: User Reconnects to Different Instance

**Priority:** High
**Type:** Unit

**Preconditions:**
- Alice was on notif-07
- Alice reconnects to notif-12

**Steps:**
1. Receive: `alice:offline`
2. Receive: `alice:online:notif-12`

**Expected Results:**
- Old routing (notif-07) removed
- New routing (notif-12) added
- No messages lost during transition

---

### TC-FAN-PRES-004: Presence TTL Expiry

**Priority:** High
**Type:** Integration

**Preconditions:**
- User's presence key expires in Redis (TTL timeout)

**Steps:**
1. Redis keyspace notification for expiry

**Expected Results:**
- User treated as offline
- Routing table updated

---

## 4. Membership Changes

### TC-FAN-MEMB-001: User Joins Channel (Online)

**Priority:** High
**Type:** Unit, Integration

**Preconditions:**
- Alice is online on notif-07
- Alice not member of ch_new

**Steps:**
1. Membership event: Alice joins ch_new

**Expected Results:**
- ch_new routing includes notif-07
- Alice's channel list includes ch_new
- Membership cache invalidated

**Go Test:**
```go
func TestMembershipChange_UserJoinsChannel_UpdatesRouting(t *testing.T) {
    rt := NewRoutingTable()
    // Alice already online with some channels
    rt.AddUser("usr_alice", "notif-07", &MockMembershipCache{
        Memberships: map[string][]string{
            "usr_alice": {"ch_general"},
        },
    })

    handler := NewMembershipHandler(rt)
    handler.HandleMemberJoin(MembershipEvent{
        ChannelID: "ch_new",
        UserID:    "usr_alice",
    })

    // Verify ch_new now routes to notif-07
    entry := rt.GetChannel("ch_new")
    assert.Contains(t, entry.Instances, "notif-07")
}
```

---

### TC-FAN-MEMB-002: User Joins Channel (Offline)

**Priority:** Medium
**Type:** Unit

**Preconditions:**
- Alice is offline

**Steps:**
1. Membership event: Alice joins ch_new

**Expected Results:**
- Membership cache invalidated
- No routing table update (Alice offline)
- When Alice comes online, new channel included

---

### TC-FAN-MEMB-003: User Leaves Channel

**Priority:** High
**Type:** Unit

**Steps:**
1. Membership event: Alice leaves ch_general

**Expected Results:**
- Alice removed from ch_general routing
- Alice's channel list updated
- Messages to ch_general no longer delivered to Alice

---

### TC-FAN-MEMB-004: User Removed from Channel (Kicked)

**Priority:** High
**Type:** Unit

**Expected Results:**
- Same as leave, but immediate
- Admin action logged

---

## 5. Concurrency

### TC-FAN-CONC-001: Concurrent Routing Table Reads During Write

**Priority:** Critical
**Type:** Unit

**Steps:**
1. Start 1000 concurrent channel lookups
2. Simultaneously add/remove users

**Expected Results:**
- No data races
- No panics
- Reads see consistent state

**Go Test:**
```go
func TestRoutingTable_ConcurrentReadWrite(t *testing.T) {
    rt := NewRoutingTable()
    rt.SetChannelInstances("ch_general", []string{"notif-07"})

    var wg sync.WaitGroup
    errors := make(chan error, 100)

    // Concurrent readers
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                entry := rt.GetChannel("ch_general")
                if entry == nil {
                    // Channel might be temporarily empty during updates
                    continue
                }
            }
        }()
    }

    // Concurrent writers
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            userID := fmt.Sprintf("usr_%d", id)
            for j := 0; j < 100; j++ {
                rt.AddUser(userID, "notif-07", &MockMembershipCache{
                    Memberships: map[string][]string{userID: {"ch_general"}},
                })
                rt.RemoveUser(userID)
            }
        }(i)
    }

    wg.Wait()
    close(errors)

    for err := range errors {
        t.Errorf("concurrent error: %v", err)
    }
}
```

---

### TC-FAN-CONC-002: Lock Contention Under Load

**Priority:** High
**Type:** Performance

**Steps:**
1. 50K msg/sec event processing
2. 1K presence changes/sec

**Expected Results:**
- Lock wait time p99 < 1ms
- No degradation in throughput

---

## 6. Recovery

### TC-FAN-REC-001: Cold Start Recovery

**Priority:** Critical
**Type:** Integration

**Preconditions:**
- Fan-Out Service restarts
- Routing table is empty

**Steps:**
1. Service starts
2. Query all online users from Redis
3. Rebuild routing table

**Expected Results:**
- Routing table rebuilt within 60 seconds
- No messages lost during recovery
- Consumer catches up on missed events

---

### TC-FAN-REC-002: Redis Connection Loss

**Priority:** High
**Type:** Integration

**Steps:**
1. Redis becomes unavailable
2. Presence updates fail

**Expected Results:**
- Circuit breaker opens
- Routing table serves stale data
- Reconnects when Redis available
- Logs errors appropriately

---

### TC-FAN-REC-003: NATS Reconnection

**Priority:** High
**Type:** Integration

**Steps:**
1. NATS node fails
2. Client reconnects to different node

**Expected Results:**
- Consumer resumes from last ack'd position
- No duplicate processing
- No message loss

---

## 7. Performance

### TC-FAN-PERF-001: Routing Lookup Latency

**Priority:** Critical
**Type:** Performance

**Benchmark:**
```go
func BenchmarkRoutingTableLookup(b *testing.B) {
    rt := NewRoutingTable()
    // Pre-populate with 500K channels
    for i := 0; i < 500000; i++ {
        rt.SetChannelInstances(fmt.Sprintf("ch_%d", i), []string{"notif-07"})
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        rt.GetChannel("ch_250000")
    }
}
```

**Expected Results:**
- < 100ns per lookup
- O(1) complexity

---

### TC-FAN-PERF-002: Event Processing Throughput

**Priority:** Critical
**Type:** Performance

**Test:**
- Publish 50K events/sec to JetStream
- Measure fan-out processing rate

**Expected Results:**
- Processing rate ≥ 50K events/sec
- Consumer lag < 1K messages
- Latency p99 < 10ms

---

### TC-FAN-PERF-003: Memory Usage at Scale

**Priority:** High
**Type:** Performance

**Test:**
- 100K online users
- 500K active channels
- Average 1K channels per user

**Expected Results:**
- Memory usage < 3GB
- No memory growth over time (soak test)

---

### TC-FAN-PERF-004: Presence Update Latency

**Priority:** High
**Type:** Performance

**Test:**
- User with 100K channel memberships comes online

**Expected Results:**
- Routing table update < 100ms
- No blocking of event processing

---

## Test Execution Summary

| Category | Total | Critical | High | Medium | Low |
|----------|-------|----------|------|--------|-----|
| Routing Table Operations | 5 | 3 | 2 | 0 | 0 |
| Event Processing | 6 | 2 | 4 | 0 | 0 |
| Presence Handling | 4 | 2 | 2 | 0 | 0 |
| Membership Changes | 4 | 0 | 3 | 1 | 0 |
| Concurrency | 2 | 1 | 1 | 0 | 0 |
| Recovery | 3 | 1 | 2 | 0 | 0 |
| Performance | 4 | 2 | 2 | 0 | 0 |
| **Total** | **28** | **11** | **16** | **1** | **0** |
