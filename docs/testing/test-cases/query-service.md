# Test Cases: Query Service

**Service:** Query Service
**Version:** 1.0
**Last Updated:** 2026-02-02

---

## Overview

The Query Service handles all read operations: message history, channel lists, user profiles, search results. It reads from Cassandra (messages), MongoDB (metadata), and Elasticsearch (search).

---

## Test Categories

1. [Message History](#1-message-history)
2. [Channel Operations](#2-channel-operations)
3. [User Operations](#3-user-operations)
4. [Search](#4-search)
5. [Pagination](#5-pagination)
6. [Caching](#6-caching)
7. [Performance](#7-performance)

---

## 1. Message History

### TC-QRY-HIST-001: Get Channel Messages (Default Pagination)

**Priority:** Critical
**Type:** Unit, Integration

**Request:**
```
GET /api/v1/channels.history?channel_id=ch_general
```

**Expected Results:**
- Status: 200 OK
- Returns last 50 messages (default)
- Messages ordered by timestamp descending
- Includes sender info, timestamps, reactions

**Go Test:**
```go
func TestGetHistory_DefaultPagination_ReturnsLast50(t *testing.T) {
    // Arrange
    repo := &MockMessageRepo{
        Messages: generateMessages(100),
    }
    handler := NewQueryHandler(repo)

    req := HistoryRequest{ChannelID: "ch_general"}

    // Act
    resp, err := handler.GetHistory(context.Background(), req)

    // Assert
    require.NoError(t, err)
    assert.Len(t, resp.Messages, 50)
    assert.True(t, resp.HasMore)
    // Verify descending order
    for i := 1; i < len(resp.Messages); i++ {
        assert.True(t, resp.Messages[i-1].Timestamp.After(resp.Messages[i].Timestamp))
    }
}
```

---

### TC-QRY-HIST-002: Get Channel Messages with Limit

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/channels.history?channel_id=ch_general&limit=20
```

**Expected Results:**
- Returns exactly 20 messages
- `has_more: true` if more messages exist

---

### TC-QRY-HIST-003: Get Channel Messages with Cursor (Before)

**Priority:** High
**Type:** Unit, Integration

**Request:**
```
GET /api/v1/channels.history?channel_id=ch_general&before=msg_123&limit=50
```

**Expected Results:**
- Returns 50 messages older than `msg_123`
- Does not include `msg_123` itself

---

### TC-QRY-HIST-004: Get Channel Messages with Cursor (After)

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/channels.history?channel_id=ch_general&after=msg_123&limit=50
```

**Expected Results:**
- Returns 50 messages newer than `msg_123`
- Used for catching up on missed messages

---

### TC-QRY-HIST-005: Get Channel Messages - Empty Channel

**Priority:** Medium
**Type:** Unit

**Expected Results:**
- Status: 200 OK
- Empty messages array
- `has_more: false`

---

### TC-QRY-HIST-006: Get Channel Messages - Unauthorized

**Priority:** Critical
**Type:** Unit

**Preconditions:**
- User is not member of channel

**Expected Results:**
- Status: 403 Forbidden
- Error: `not_channel_member`

---

### TC-QRY-HIST-007: Get Thread Messages

**Priority:** High
**Type:** Unit, Integration

**Request:**
```
GET /api/v1/threads.history?thread_id=msg_parent
```

**Expected Results:**
- Returns thread replies only
- Includes parent message summary
- Ordered by timestamp ascending (oldest first)

---

### TC-QRY-HIST-008: Get Message by ID

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/messages.get?message_id=msg_123
```

**Expected Results:**
- Returns single message with full details
- Includes reactions, thread info if applicable

---

### TC-QRY-HIST-009: Get Pinned Messages

**Priority:** Medium
**Type:** Unit

**Request:**
```
GET /api/v1/channels.pinned?channel_id=ch_general
```

**Expected Results:**
- Returns only pinned messages
- Ordered by pin timestamp

---

## 2. Channel Operations

### TC-QRY-CH-001: List User's Channels

**Priority:** Critical
**Type:** Unit, Integration

**Request:**
```
GET /api/v1/channels.list
```

**Expected Results:**
- Returns all channels user is member of
- Includes unread counts
- Includes last message preview
- Ordered by last activity

**Go Test:**
```go
func TestListChannels_ReturnsUserChannels(t *testing.T) {
    repo := &MockChannelRepo{
        Memberships: map[string][]string{
            "usr_alice": {"ch_general", "ch_dev", "dm_alice_bob"},
        },
    }
    handler := NewQueryHandler(repo)

    ctx := contextWithUser(context.Background(), "usr_alice")
    resp, err := handler.ListChannels(ctx)

    require.NoError(t, err)
    assert.Len(t, resp.Channels, 3)
}
```

---

### TC-QRY-CH-002: Get Channel Info

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/channels.info?channel_id=ch_general
```

**Expected Results:**
- Returns channel details (name, description, type)
- Includes member count
- Includes user's membership status

---

### TC-QRY-CH-003: List Channel Members

**Priority:** High
**Type:** Unit, Integration

**Request:**
```
GET /api/v1/channels.members?channel_id=ch_general
```

**Expected Results:**
- Returns paginated member list
- Includes online status
- Includes role (owner, admin, member)

---

### TC-QRY-CH-004: Get Channel Unread Count

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/channels.unread?channel_id=ch_general
```

**Expected Results:**
- Returns unread message count
- Returns unread mention count
- Returns last read message ID

---

### TC-QRY-CH-005: Search Public Channels

**Priority:** Medium
**Type:** Unit

**Request:**
```
GET /api/v1/channels.search?query=engineering
```

**Expected Results:**
- Returns matching public channels
- Does not include private channels
- Ranked by relevance

---

## 3. User Operations

### TC-QRY-USR-001: Get User Profile

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/users.info?user_id=usr_alice
```

**Expected Results:**
- Returns user profile (name, avatar, status)
- Includes online presence
- Includes custom status message

---

### TC-QRY-USR-002: Get Current User

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/users.me
```

**Expected Results:**
- Returns authenticated user's profile
- Includes preferences
- Includes notification settings

---

### TC-QRY-USR-003: Search Users

**Priority:** Medium
**Type:** Unit

**Request:**
```
GET /api/v1/users.search?query=ali
```

**Expected Results:**
- Returns matching users
- Search by name, email, username
- Respects privacy settings

---

### TC-QRY-USR-004: Get User Presence

**Priority:** High
**Type:** Unit

**Request:**
```
GET /api/v1/users.presence?user_ids=usr_alice,usr_bob
```

**Expected Results:**
- Returns presence for each user
- Status: online, idle, away, offline
- Includes last active timestamp

---

## 4. Search

### TC-QRY-SRCH-001: Full-Text Search Messages

**Priority:** High
**Type:** Unit, Integration

**Request:**
```
GET /api/v1/search.messages?query=kubernetes deployment
```

**Expected Results:**
- Returns messages containing search terms
- Results ranked by relevance
- Includes channel info for each result
- Highlights matching terms

**Go Test:**
```go
func TestSearchMessages_ReturnsRelevantResults(t *testing.T) {
    esClient := &MockElasticsearch{
        Results: []SearchHit{
            {ID: "msg_1", Score: 10.5, Source: map[string]interface{}{"text": "kubernetes deployment failed"}},
            {ID: "msg_2", Score: 8.2, Source: map[string]interface{}{"text": "deployment to kubernetes cluster"}},
        },
    }
    handler := NewQueryHandler(nil, esClient)

    resp, err := handler.SearchMessages(context.Background(), SearchRequest{
        Query: "kubernetes deployment",
        Limit: 20,
    })

    require.NoError(t, err)
    assert.Len(t, resp.Results, 2)
    assert.Equal(t, "msg_1", resp.Results[0].MessageID) // Higher score first
}
```

---

### TC-QRY-SRCH-002: Search with Channel Filter

**Priority:** Medium
**Type:** Unit

**Request:**
```
GET /api/v1/search.messages?query=bug&channel_id=ch_engineering
```

**Expected Results:**
- Only searches within specified channel
- User must have access to channel

---

### TC-QRY-SRCH-003: Search with Date Range

**Priority:** Medium
**Type:** Unit

**Request:**
```
GET /api/v1/search.messages?query=release&after=2026-01-01&before=2026-02-01
```

**Expected Results:**
- Only returns messages within date range
- Inclusive of boundary dates

---

### TC-QRY-SRCH-004: Search with Sender Filter

**Priority:** Medium
**Type:** Unit

**Request:**
```
GET /api/v1/search.messages?query=update&from=usr_alice
```

**Expected Results:**
- Only returns messages from specified sender

---

### TC-QRY-SRCH-005: Search - No Results

**Priority:** Medium
**Type:** Unit

**Expected Results:**
- Status: 200 OK
- Empty results array
- `total_count: 0`

---

### TC-QRY-SRCH-006: Search - Unauthorized Channel Access

**Priority:** High
**Type:** Unit

**Preconditions:**
- Search results include messages from private channel user doesn't have access to

**Expected Results:**
- Private channel messages filtered out
- Only accessible messages returned

---

## 5. Pagination

### TC-QRY-PAGE-001: Cursor-Based Pagination

**Priority:** High
**Type:** Unit

**Steps:**
1. Request first page: `GET /api/v1/channels.history?channel_id=ch_general&limit=20`
2. Get `next_cursor` from response
3. Request next page: `GET /api/v1/channels.history?channel_id=ch_general&limit=20&cursor={next_cursor}`

**Expected Results:**
- No duplicate messages between pages
- No missing messages between pages
- Stable during concurrent writes

---

### TC-QRY-PAGE-002: Invalid Cursor

**Priority:** Medium
**Type:** Unit

**Request:**
```
GET /api/v1/channels.history?channel_id=ch_general&cursor=invalid
```

**Expected Results:**
- Status: 400 Bad Request
- Error: `invalid_cursor`

---

### TC-QRY-PAGE-003: Expired Cursor

**Priority:** Low
**Type:** Unit

**Preconditions:**
- Cursor is older than retention period

**Expected Results:**
- Status: 400 Bad Request
- Error: `cursor_expired`

---

### TC-QRY-PAGE-004: Limit Boundaries

**Priority:** Medium
**Type:** Unit

**Test Cases:**
- `limit=0` → Uses default (50)
- `limit=-1` → Error: invalid_limit
- `limit=1001` → Capped at 1000

---

## 6. Caching

### TC-QRY-CACHE-001: Cache Hit for Channel Info

**Priority:** High
**Type:** Integration

**Steps:**
1. Request channel info
2. Request same channel info again

**Expected Results:**
- Second request served from cache
- Cache header indicates hit
- No database query on second request

---

### TC-QRY-CACHE-002: Cache Invalidation on Update

**Priority:** High
**Type:** Integration

**Steps:**
1. Request channel info (cache populated)
2. Update channel name
3. Request channel info again

**Expected Results:**
- Third request returns updated data
- Cache invalidated by update event

---

### TC-QRY-CACHE-003: Cache Miss Fallback

**Priority:** High
**Type:** Integration

**Preconditions:**
- Cache is empty or unavailable

**Expected Results:**
- Request served from database
- Response cached for future requests

---

## 7. Performance

### TC-QRY-PERF-001: Message History Latency

**Priority:** Critical
**Type:** Performance

**Benchmark:**
```go
func BenchmarkGetHistory(b *testing.B) {
    handler := setupHandler()
    req := HistoryRequest{ChannelID: "ch_general", Limit: 50}
    ctx := context.Background()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        handler.GetHistory(ctx, req)
    }
}
```

**Expected Results:**
- p50 < 10ms
- p99 < 50ms

---

### TC-QRY-PERF-002: Search Latency

**Priority:** High
**Type:** Performance

**Expected Results:**
- p50 < 100ms
- p99 < 500ms

---

### TC-QRY-PERF-003: Concurrent Read Load

**Priority:** High
**Type:** Performance

**Test:**
- 1000 concurrent requests for message history
- Mix of different channels

**Expected Results:**
- No errors
- Latency degradation < 2x baseline

---

## Test Execution Summary

| Category | Total | Critical | High | Medium | Low |
|----------|-------|----------|------|--------|-----|
| Message History | 9 | 2 | 5 | 2 | 0 |
| Channel Operations | 5 | 1 | 3 | 1 | 0 |
| User Operations | 4 | 0 | 3 | 1 | 0 |
| Search | 6 | 0 | 3 | 3 | 0 |
| Pagination | 4 | 0 | 2 | 1 | 1 |
| Caching | 3 | 0 | 3 | 0 | 0 |
| Performance | 3 | 1 | 2 | 0 | 0 |
| **Total** | **34** | **4** | **21** | **8** | **1** |
