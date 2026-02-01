# Last Seen & Unread Indicators

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Data Model](#2-data-model)
3. [Unread Computation](#3-unread-computation)
4. [Real-Time Badge Updates](#4-real-time-badge-updates)
5. [Bulk Unread Sync on Login](#5-bulk-unread-sync-on-login)
6. [Thread Unread Indicators](#6-thread-unread-indicators)
7. [Cross-Device Sync](#7-cross-device-sync)

---

## 1. Requirements

- **Unread dot:** Visual indicator on channels that have messages the user hasn't seen
- **Unread count:** Badge showing the number of unread messages (capped at "99+")
- **Channel list ordering:** Channels sorted by most recent activity, with unread channels surfaced to the top
- **Thread unread:** Separate unread indicator for threads the user follows with new replies
- **Real-time updates:** Unread count increments live when new messages arrive in non-focused channels
- **Cross-device sync:** If the user reads on mobile, desktop should reflect this

---

## 2. Data Model

### Redis Key: `read-pointer:{user_id}:{channel_id}`

The primary data structure for unread computation:

```
Key:    read-pointer:{user_id}:{channel_id}
Type:   Hash
Value:  {
  "last_read_id": "msg_01HZ4L...",
  "last_read_at": "2026-02-01T10:35:00Z"
}
TTL: none (persistent)
```

### Redis Key: `channel-latest:{channel_id}`

Tracks the most recent message in each channel:

```
Key:    channel-latest:{channel_id}
Type:   Hash
Value:  {
  "latest_id": "msg_01HZ5M...",
  "latest_at": "2026-02-01T10:40:00Z",
  "latest_sender": "usr_carol",
  "latest_preview": "Has anyone seen the...",
  "message_seq": "142857"
}
TTL: none (persistent)
```

**Updated by:** The Message Writer Worker, after writing each message to Cassandra, also updates the `channel-latest` Redis key.

### Redis Key: `thread-read:{user_id}:{thread_id}`

Separate read pointers for thread-level unread:

```
Key:    thread-read:{user_id}:{thread_id}
Type:   Hash
Value:  {
  "last_read_id": "msg_01HZ4R...",
  "last_read_at": "2026-02-01T10:36:00Z"
}
TTL: 30 days (EXPIRE)
```

---

## 3. Unread Computation

### Simple Unread Check (has unread? yes/no)

```go
func hasUnread(userID, channelID string) bool {
    readPointer, _ := redis.HGetAll("read-pointer:" + userID + ":" + channelID)
    channelLatest, _ := redis.HGetAll("channel-latest:" + channelID)

    if len(readPointer) == 0 {
        return len(channelLatest) > 0  // never read → unread if any messages exist
    }

    return channelLatest["latest_id"] > readPointer["last_read_id"]
}
```

**Cost:** 2 Redis HGETALL commands — sub-millisecond.

### Unread Count (badge number)

Computing exact unread count requires querying Cassandra:

```sql
-- Count messages after user's last-read position
SELECT COUNT(*) FROM messages
WHERE channel_id = ?
  AND bucket IN (?)           -- buckets between last_read_at and now
  AND message_id > ?          -- last_read_id as cursor
LIMIT 100;                    -- cap at 99+ for UI
```

### Optimization: Client-Side Tracking

For channels where the user is actively receiving events, the **client tracks unread count locally**: it increments a local counter for each `message.new` WebSocket event in non-focused channels and resets to 0 when the user opens the channel.

Server-side Cassandra queries are only needed:
1. On initial login (Tier 3/4 reconnection)
2. When a channel is opened for the first time in a session

In practice, the client handles 90%+ of unread count updates in-memory.

### Per-Channel Sequence Numbers (Optional)

For exact unread counts without Cassandra queries:

```
Bucket: channel-latest
  Value: { ..., "message_seq": 142857 }

Bucket: read-pointers
  Value: { ..., "last_read_seq": 142830 }
```

Unread count = `channel_latest.message_seq - read_pointer.last_read_seq`

**Challenge:** The per-channel sequence must be atomically incremented. Batch sequence allocation mitigates contention.

---

## 4. Real-Time Badge Updates

When a new message arrives in a channel the user is NOT focused on:

1. Fan-Out Service delivers the event to the user's Notification Service instance
2. Notification Service checks local state: is this user focused on this channel?
   - **Yes:** Don't increment unread (user is actively viewing)
   - **No:** Send a `message.new` event over WebSocket. The client increments local unread count.

### Client Focus Tracking

```json
// Client → Server: user switches to channel
{"type": "channel.focus", "channel_id": "ch_general"}

// Client → Server: user switches away
{"type": "channel.blur", "channel_id": "ch_general"}
```

The Notification Service tracks the focused channel per connection. When focused, it auto-sends `mark_read` for messages in that channel, updating the `read-pointers` KV.

---

## 5. Bulk Unread Sync on Login

When a user connects and needs unread status for all their channels:

```go
func bulkUnreadSync(userID string, channelIDs []string) []UnreadStatus {
    // 1. Bulk read user's read pointers
    readPointers := kvBulkGet("read-pointers", "user/"+userID+"/channel/*")

    // 2. Bulk read channel latest
    channelLatest := kvBulkGet("channel-latest", channelIDs)

    // 3. Compare in memory
    results := make([]UnreadStatus, 0)
    for _, ch := range channelIDs {
        rp := readPointers[ch]
        cl := channelLatest[ch]
        if cl != nil && (rp == nil || cl.LatestID > rp.LastReadID) {
            results = append(results, UnreadStatus{
                ChannelID: ch,
                Unread:    true,
                LatestAt:  cl.LatestAt,
                Preview:   cl.LatestPreview,
            })
        }
    }

    // 4. Sort by latest activity
    sort.Slice(results, func(i, j int) bool {
        return results[i].LatestAt.After(results[j].LatestAt)
    })

    return results
}
```

**Performance:** For a user in 100K channels, this requires ~100K Redis reads + comparisons. Redis MGET can process ~100K keys in 200–500ms. Acceptable for a login operation.

---

## 6. Thread Unread Indicators

Thread unread follows the same pattern:

1. `thread-read:{user_id}:{thread_id}` Redis key tracks last-read position per thread per user
2. When a thread reply arrives, compare with the user's thread read pointer
3. Show unread badge on the thread root in the channel timeline
4. Show unread count in the "Threads" panel

**Scope limitation:** Only compute thread unread for threads the user **follows**. This bounds the computation to ~10–100 threads per user.

---

## 7. Cross-Device Sync

When a user reads a channel on Device A, Device B should reflect this:

1. Device A sends `mark_read` → Notification Service updates `read-pointer:*` in Redis
2. The Notification Service publishes to Redis pub/sub channel `read-pointer:changes`
3. If the user is connected on another instance (Device B), that instance pushes:
   ```json
   {
     "type": "read_pointer.updated",
     "channel_id": "ch_general",
     "last_read_id": "msg_01HZ5M...",
     "last_read_at": "..."
   }
   ```
4. Device B's client resets its local unread count for that channel

**Implementation:** The Notification Service subscribes to Redis pub/sub channel `read-pointer:changes`. When a change is detected from a different instance (compare `instance_id`), it relays the update to local clients.
