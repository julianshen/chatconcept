# Test Cases: Command Service

**Service:** Command Service
**Version:** 1.0
**Last Updated:** 2026-02-02

---

## Overview

The Command Service handles all write operations: message sending, channel creation, user updates, etc. It validates input, publishes events to NATS JetStream, and returns `202 Accepted`.

**Critical Invariant:** Command Service **never** writes directly to databases. All persistence is handled by downstream workers.

---

## Test Categories

1. [Message Sending](#1-message-sending)
2. [Message Editing](#2-message-editing)
3. [Message Deletion](#3-message-deletion)
4. [Reactions](#4-reactions)
5. [Channel Management](#5-channel-management)
6. [Thread Operations](#6-thread-operations)
7. [Input Validation](#7-input-validation)
8. [Authorization](#8-authorization)
9. [Idempotency](#9-idempotency)
10. [Error Handling](#10-error-handling)

---

## 1. Message Sending

### TC-CMD-MSG-001: Send Valid Plain Text Message

**Priority:** Critical
**Type:** Unit, Integration

**Preconditions:**
- User is authenticated
- User is member of target channel

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "Hello, world!"
}
```

**Steps:**
1. POST `/api/v1/chat.sendMessage` with valid payload
2. Verify response status is `202 Accepted`
3. Verify response contains `message_id` and `correlation_id`
4. Verify event published to `messages.send.ch_general`

**Expected Results:**
- Status: 202 Accepted
- Response body contains valid `message_id`
- NATS event contains correct payload
- Event includes `sender_id`, `timestamp`, preprocessed content

**Go Test:**
```go
func TestSendMessage_ValidPlainText_PublishesEvent(t *testing.T) {
    // Arrange
    publisher := &MockPublisher{}
    authz := &MockAuthz{Allowed: true}
    handler := NewMessageHandler(publisher, authz)

    req := SendMessageRequest{
        ChannelID: "ch_general",
        Text:      "Hello, world!",
    }

    ctx := contextWithUser(context.Background(), "usr_alice")

    // Act
    resp, err := handler.SendMessage(ctx, req)

    // Assert
    require.NoError(t, err)
    assert.Equal(t, http.StatusAccepted, resp.StatusCode)
    assert.NotEmpty(t, resp.MessageID)

    require.Len(t, publisher.Published, 1)
    event := publisher.Published[0]
    assert.Equal(t, "messages.send.ch_general", event.Subject)
    assert.Equal(t, "Hello, world!", event.Payload.Text)
    assert.Equal(t, "usr_alice", event.Payload.SenderID)
}
```

---

### TC-CMD-MSG-002: Send Message with Mentions

**Priority:** High
**Type:** Unit, Integration

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "Hey @alice and @bob, check this out!"
}
```

**Expected Results:**
- Mentions extracted: `["usr_alice", "usr_bob"]`
- Event payload includes `mentions` array
- Mention usernames resolved to user IDs

**Go Test:**
```go
func TestSendMessage_WithMentions_ExtractsMentions(t *testing.T) {
    publisher := &MockPublisher{}
    userResolver := &MockUserResolver{
        Users: map[string]string{
            "alice": "usr_alice",
            "bob":   "usr_bob",
        },
    }
    handler := NewMessageHandler(publisher, nil, userResolver)

    req := SendMessageRequest{
        ChannelID: "ch_general",
        Text:      "Hey @alice and @bob, check this out!",
    }

    _, err := handler.SendMessage(context.Background(), req)

    require.NoError(t, err)
    event := publisher.Published[0].Payload
    assert.ElementsMatch(t, []string{"usr_alice", "usr_bob"}, event.Mentions)
}
```

---

### TC-CMD-MSG-003: Send Message with @here Mention

**Priority:** High
**Type:** Unit

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "Attention @here - meeting in 5 minutes"
}
```

**Expected Results:**
- `mention_here: true` in event payload
- Used for notification targeting (online members only)

---

### TC-CMD-MSG-004: Send Message with @all Mention

**Priority:** High
**Type:** Unit

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "@all Important announcement!"
}
```

**Expected Results:**
- `mention_all: true` in event payload
- Used for notification targeting (all members)

---

### TC-CMD-MSG-005: Send Message with URL (Link Preview)

**Priority:** Medium
**Type:** Unit, Integration

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "Check out https://example.com/article"
}
```

**Expected Results:**
- URL extracted from text
- Additional event published to `link_preview.fetch.{message_id}`
- Message delivered immediately; preview attached asynchronously

---

### TC-CMD-MSG-006: Send Message with Emoji Shortcodes

**Priority:** Medium
**Type:** Unit

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "Great job! :thumbsup: :rocket:"
}
```

**Expected Results:**
- Emoji shortcodes resolved to Unicode: 👍 🚀
- Original shortcodes preserved in `raw_text` field

---

### TC-CMD-MSG-007: Send Message with Code Block

**Priority:** Medium
**Type:** Unit

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "Here's the code:\n```go\nfunc main() {\n    fmt.Println(\"Hello\")\n}\n```"
}
```

**Expected Results:**
- Code block preserved in output
- Language hint (`go`) extracted for syntax highlighting
- No emoji or mention processing inside code blocks

---

### TC-CMD-MSG-008: Send Message to DM Channel

**Priority:** High
**Type:** Unit, Integration

**Test Data:**
```json
{
  "channel_id": "dm_alice_bob",
  "text": "Private message"
}
```

**Expected Results:**
- Event published to `messages.send.dm_alice_bob`
- Channel type identified as DM
- Only 2 members in recipient list

---

### TC-CMD-MSG-009: Send Message with Maximum Length

**Priority:** High
**Type:** Unit

**Test Data:**
- Text: 50,000 characters (maximum allowed)

**Expected Results:**
- Message accepted
- No truncation
- Full text preserved in event

---

### TC-CMD-MSG-010: Send Message Exceeding Maximum Length

**Priority:** High
**Type:** Unit

**Test Data:**
- Text: 50,001 characters

**Expected Results:**
- Status: 400 Bad Request
- Error: `message_too_long`
- No event published

**Go Test:**
```go
func TestSendMessage_ExceedsMaxLength_ReturnsError(t *testing.T) {
    handler := NewMessageHandler(&MockPublisher{}, nil)

    req := SendMessageRequest{
        ChannelID: "ch_general",
        Text:      strings.Repeat("a", 50001),
    }

    _, err := handler.SendMessage(context.Background(), req)

    assert.ErrorIs(t, err, ErrMessageTooLong)
}
```

---

## 2. Message Editing

### TC-CMD-EDIT-001: Edit Own Message

**Priority:** High
**Type:** Unit, Integration

**Preconditions:**
- Original message exists
- User is the message sender

**Test Data:**
```json
{
  "message_id": "msg_123",
  "text": "Updated message text"
}
```

**Expected Results:**
- Status: 202 Accepted
- Event published to `messages.edit.{channel_id}`
- Event includes `edited_at` timestamp
- Event includes both `old_text` and `new_text`

---

### TC-CMD-EDIT-002: Edit Message Not Owned

**Priority:** High
**Type:** Unit

**Preconditions:**
- Message exists but was sent by different user

**Expected Results:**
- Status: 403 Forbidden
- Error: `not_message_owner`
- No event published

---

### TC-CMD-EDIT-003: Edit Message After Time Limit

**Priority:** Medium
**Type:** Unit

**Preconditions:**
- Message is older than edit time limit (e.g., 24 hours)

**Expected Results:**
- Status: 400 Bad Request
- Error: `edit_time_expired`

---

### TC-CMD-EDIT-004: Edit Nonexistent Message

**Priority:** High
**Type:** Unit

**Expected Results:**
- Status: 404 Not Found
- Error: `message_not_found`

---

## 3. Message Deletion

### TC-CMD-DEL-001: Delete Own Message

**Priority:** High
**Type:** Unit, Integration

**Expected Results:**
- Status: 202 Accepted
- Event published to `messages.delete.{channel_id}`
- Event includes `deleted_by` user ID

---

### TC-CMD-DEL-002: Admin Delete Any Message

**Priority:** High
**Type:** Unit, Integration

**Preconditions:**
- User has admin role in channel

**Expected Results:**
- Status: 202 Accepted
- `deleted_by` shows admin user ID
- Audit event logged

---

### TC-CMD-DEL-003: Delete Message Not Permitted

**Priority:** High
**Type:** Unit

**Preconditions:**
- User is not sender and not admin

**Expected Results:**
- Status: 403 Forbidden
- Error: `delete_not_permitted`

---

## 4. Reactions

### TC-CMD-REACT-001: Add Reaction to Message

**Priority:** Medium
**Type:** Unit, Integration

**Test Data:**
```json
{
  "message_id": "msg_123",
  "emoji": "👍"
}
```

**Expected Results:**
- Status: 202 Accepted
- Event published to `messages.react.{channel_id}`
- Event includes `emoji`, `user_id`, `action: add`

---

### TC-CMD-REACT-002: Remove Reaction from Message

**Priority:** Medium
**Type:** Unit

**Test Data:**
```json
{
  "message_id": "msg_123",
  "emoji": "👍",
  "action": "remove"
}
```

**Expected Results:**
- Status: 202 Accepted
- Event includes `action: remove`

---

### TC-CMD-REACT-003: Add Duplicate Reaction

**Priority:** Low
**Type:** Unit

**Preconditions:**
- User already reacted with same emoji

**Expected Results:**
- Status: 200 OK (idempotent)
- No duplicate event published

---

### TC-CMD-REACT-004: Add Reaction with Invalid Emoji

**Priority:** Medium
**Type:** Unit

**Test Data:**
```json
{
  "message_id": "msg_123",
  "emoji": "invalid"
}
```

**Expected Results:**
- Status: 400 Bad Request
- Error: `invalid_emoji`

---

## 5. Channel Management

### TC-CMD-CH-001: Create Public Channel

**Priority:** High
**Type:** Unit, Integration

**Test Data:**
```json
{
  "name": "engineering",
  "type": "public",
  "description": "Engineering team discussions"
}
```

**Expected Results:**
- Status: 202 Accepted
- Event published to `channels.create`
- Channel ID generated with `ch_` prefix
- Creator automatically added as member and owner

---

### TC-CMD-CH-002: Create Private Channel

**Priority:** High
**Type:** Unit, Integration

**Test Data:**
```json
{
  "name": "secret-project",
  "type": "private",
  "members": ["usr_alice", "usr_bob"]
}
```

**Expected Results:**
- Status: 202 Accepted
- Initial members added
- Channel not discoverable in public list

---

### TC-CMD-CH-003: Create Channel with Duplicate Name

**Priority:** Medium
**Type:** Unit

**Preconditions:**
- Channel with same name exists

**Expected Results:**
- Status: 409 Conflict
- Error: `channel_name_exists`

---

### TC-CMD-CH-004: Add Member to Channel

**Priority:** High
**Type:** Unit, Integration

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "user_id": "usr_carol"
}
```

**Expected Results:**
- Status: 202 Accepted
- Event published to `channels.member.join.{channel_id}`
- Fan-Out Service updates routing table

---

### TC-CMD-CH-005: Remove Member from Channel

**Priority:** High
**Type:** Unit, Integration

**Expected Results:**
- Status: 202 Accepted
- Event published to `channels.member.leave.{channel_id}`
- User loses access immediately

---

### TC-CMD-CH-006: Update Channel Settings

**Priority:** Medium
**Type:** Unit

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "name": "general-new",
  "description": "Updated description"
}
```

**Expected Results:**
- Status: 202 Accepted
- Event published to `channels.update.{channel_id}`
- Only changed fields included in event

---

## 6. Thread Operations

### TC-CMD-THR-001: Reply to Thread

**Priority:** High
**Type:** Unit, Integration

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "thread_id": "msg_parent",
  "text": "Thread reply"
}
```

**Expected Results:**
- Status: 202 Accepted
- Event published to `messages.thread_reply.{channel_id}`
- Event includes `thread_id` and `also_send_to_channel: false`

---

### TC-CMD-THR-002: Reply to Thread with Also Send to Channel

**Priority:** High
**Type:** Unit, Integration

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "thread_id": "msg_parent",
  "text": "Thread reply (also in channel)",
  "also_send_to_channel": true
}
```

**Expected Results:**
- Event includes `also_send_to_channel: true`
- Message appears in both thread and main channel

---

### TC-CMD-THR-003: Reply to Nonexistent Thread

**Priority:** High
**Type:** Unit

**Expected Results:**
- Status: 404 Not Found
- Error: `thread_not_found`

---

## 7. Input Validation

### TC-CMD-VAL-001: Empty Message Text

**Priority:** Critical
**Type:** Unit

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": ""
}
```

**Expected Results:**
- Status: 400 Bad Request
- Error: `empty_message`

**Go Test:**
```go
func TestValidation_EmptyText_ReturnsError(t *testing.T) {
    tests := []struct {
        name    string
        text    string
        wantErr error
    }{
        {"empty string", "", ErrEmptyMessage},
        {"whitespace only", "   ", ErrEmptyMessage},
        {"newlines only", "\n\n", ErrEmptyMessage},
        {"valid text", "hello", nil},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateMessageText(tt.text)
            if !errors.Is(err, tt.wantErr) {
                t.Errorf("got %v, want %v", err, tt.wantErr)
            }
        })
    }
}
```

---

### TC-CMD-VAL-002: Missing Channel ID

**Priority:** Critical
**Type:** Unit

**Test Data:**
```json
{
  "text": "Hello"
}
```

**Expected Results:**
- Status: 400 Bad Request
- Error: `missing_channel_id`

---

### TC-CMD-VAL-003: Invalid Channel ID Format

**Priority:** High
**Type:** Unit

**Test Data:**
```json
{
  "channel_id": "invalid-format",
  "text": "Hello"
}
```

**Expected Results:**
- Status: 400 Bad Request
- Error: `invalid_channel_id`

---

### TC-CMD-VAL-004: XSS Attempt in Message

**Priority:** Critical
**Type:** Unit, Security

**Test Data:**
```json
{
  "channel_id": "ch_general",
  "text": "<script>alert('xss')</script>"
}
```

**Expected Results:**
- Status: 202 Accepted (message is stored)
- HTML escaped in output: `&lt;script&gt;...`
- No script execution on client render

---

### TC-CMD-VAL-005: SQL Injection Attempt

**Priority:** Critical
**Type:** Unit, Security

**Test Data:**
```json
{
  "channel_id": "ch_general'; DROP TABLE messages;--",
  "text": "Hello"
}
```

**Expected Results:**
- Status: 400 Bad Request
- Error: `invalid_channel_id`
- No SQL executed

---

## 8. Authorization

### TC-CMD-AUTH-001: Send Message Without Authentication

**Priority:** Critical
**Type:** Unit, Integration

**Preconditions:**
- No JWT token provided

**Expected Results:**
- Status: 401 Unauthorized
- Error: `missing_authentication`

---

### TC-CMD-AUTH-002: Send Message with Expired Token

**Priority:** Critical
**Type:** Unit, Integration

**Expected Results:**
- Status: 401 Unauthorized
- Error: `token_expired`

---

### TC-CMD-AUTH-003: Send Message to Non-Member Channel

**Priority:** Critical
**Type:** Unit, Integration

**Preconditions:**
- User is not a member of the target channel

**Expected Results:**
- Status: 403 Forbidden
- Error: `not_channel_member`

---

### TC-CMD-AUTH-004: Send Message to Archived Channel

**Priority:** High
**Type:** Unit

**Preconditions:**
- Channel is archived

**Expected Results:**
- Status: 403 Forbidden
- Error: `channel_archived`

---

## 9. Idempotency

### TC-CMD-IDEM-001: Duplicate Message Send (Same Idempotency Key)

**Priority:** High
**Type:** Unit, Integration

**Steps:**
1. Send message with `Idempotency-Key: abc123`
2. Send identical message with same key

**Expected Results:**
- First request: 202 Accepted, event published
- Second request: 202 Accepted, same `message_id`, no duplicate event

**Go Test:**
```go
func TestIdempotency_DuplicateRequest_ReturnsExisting(t *testing.T) {
    publisher := &MockPublisher{}
    cache := NewIdempotencyCache()
    handler := NewMessageHandler(publisher, nil, cache)

    req := SendMessageRequest{
        ChannelID: "ch_general",
        Text:      "Hello",
    }
    ctx := contextWithIdempotencyKey(context.Background(), "key123")

    // First request
    resp1, err1 := handler.SendMessage(ctx, req)
    require.NoError(t, err1)

    // Second request with same key
    resp2, err2 := handler.SendMessage(ctx, req)
    require.NoError(t, err2)

    // Same message ID returned
    assert.Equal(t, resp1.MessageID, resp2.MessageID)
    // Only one event published
    assert.Len(t, publisher.Published, 1)
}
```

---

### TC-CMD-IDEM-002: NATS Message Deduplication

**Priority:** High
**Type:** Integration

**Steps:**
1. Publish event with `Nats-Msg-Id: evt_123`
2. Publish same event with same ID

**Expected Results:**
- NATS acknowledges both
- Only one event stored in stream

---

## 10. Error Handling

### TC-CMD-ERR-001: NATS Unavailable

**Priority:** Critical
**Type:** Integration

**Preconditions:**
- NATS cluster is down

**Expected Results:**
- Status: 503 Service Unavailable
- Error: `event_bus_unavailable`
- Circuit breaker opens after repeated failures

---

### TC-CMD-ERR-002: Rate Limit Exceeded

**Priority:** High
**Type:** Unit, Integration

**Preconditions:**
- User has sent too many messages in time window

**Expected Results:**
- Status: 429 Too Many Requests
- Header: `Retry-After: 60`
- Error: `rate_limit_exceeded`

---

### TC-CMD-ERR-003: Request Timeout

**Priority:** High
**Type:** Integration

**Steps:**
1. Simulate slow NATS publish (>5s)

**Expected Results:**
- Status: 504 Gateway Timeout
- Error: `request_timeout`
- No partial state left

---

## Test Execution Summary

| Category | Total | Critical | High | Medium | Low |
|----------|-------|----------|------|--------|-----|
| Message Sending | 10 | 2 | 5 | 3 | 0 |
| Message Editing | 4 | 0 | 3 | 1 | 0 |
| Message Deletion | 3 | 0 | 3 | 0 | 0 |
| Reactions | 4 | 0 | 0 | 3 | 1 |
| Channel Management | 6 | 0 | 4 | 2 | 0 |
| Thread Operations | 3 | 0 | 3 | 0 | 0 |
| Input Validation | 5 | 3 | 1 | 0 | 1 |
| Authorization | 4 | 3 | 1 | 0 | 0 |
| Idempotency | 2 | 0 | 2 | 0 | 0 |
| Error Handling | 3 | 1 | 2 | 0 | 0 |
| **Total** | **44** | **9** | **24** | **9** | **2** |
