# API Reference

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [API Design Principles](#2-api-design-principles)
3. [Authentication APIs](#3-authentication-apis)
4. [Chat APIs](#4-chat-apis)
5. [Channel APIs](#5-channel-apis)
6. [Direct Message APIs](#6-direct-message-apis)
7. [User APIs](#7-user-apis)
8. [Team APIs](#8-team-apis)
9. [Thread APIs](#9-thread-apis)
10. [File APIs](#10-file-apis)
11. [Reaction APIs](#11-reaction-apis)
12. [Search APIs](#12-search-apis)
13. [Push APIs](#13-push-apis)
14. [Bot APIs](#14-bot-apis)
15. [Admin APIs](#15-admin-apis)
16. [WebSocket Events](#16-websocket-events)
17. [Migration Notes](#17-migration-notes)

---

## 1. Overview

This document provides a comprehensive API reference mapping Rocket.Chat compatible endpoints to the new event-driven architecture. The API maintains backward compatibility while leveraging the new CQRS pattern.

### Architecture Mapping

| HTTP Method | Routes To | Response |
|-------------|-----------|----------|
| `GET` | Query Service | Immediate (from Cassandra/MongoDB/ES) |
| `POST/PUT/DELETE` | Command Service | `202 Accepted` with correlation ID |
| `WebSocket` | Notification Service | Real-time events |

### Base URL

```
https://chat.example.com/api/v1/
```

### Response Format

**Success:**
```json
{
  "success": true,
  "data": { ... }
}
```

**Async Command (202 Accepted):**
```json
{
  "success": true,
  "correlation_id": "corr_01HZ3K...",
  "message": "Request accepted"
}
```

**Error:**
```json
{
  "success": false,
  "error": "error_code",
  "message": "Human readable message"
}
```

---

## 2. API Design Principles

### 2.1 CQRS Pattern

| Operation | Service | Behavior |
|-----------|---------|----------|
| **Query** (GET) | Query Service | Synchronous read from data stores |
| **Command** (POST/PUT/DELETE) | Command Service | Async publish to NATS, return 202 |

### 2.2 Eventual Consistency

Write operations return `202 Accepted` immediately. The actual persistence happens asynchronously. Clients should:
1. Use the `correlation_id` for tracking
2. Rely on WebSocket events for confirmation
3. Expect brief read-after-write delays (~50ms)

### 2.3 Authentication

All endpoints (except auth) require authentication:

```http
Authorization: Bearer <access_token>
```

Or for bots:
```http
Authorization: Bearer xbot-<bot_token>
```

---

## 3. Authentication APIs

### 3.1 Login (OIDC)

Initiate OIDC login flow. **New architecture uses Keycloak exclusively.**

| | |
|---|---|
| **Endpoint** | `GET /api/v1/auth/login` |
| **Routes To** | Auth Service |
| **Auth Required** | No |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `redirect` | string | No | Post-login redirect URL |

**Response:**
```json
{
  "success": true,
  "auth_url": "https://auth.example.com/realms/chat/protocol/openid-connect/auth?...",
  "state": "state_abc123"
}
```

**Migration Note:** Replaces `login` with username/password. All authentication now via Keycloak OIDC.

---

### 3.2 OAuth Callback

Handle OIDC callback after user authentication.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/auth/callback` |
| **Routes To** | Auth Service |
| **Auth Required** | No |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `code` | string | Yes | Authorization code |
| `state` | string | Yes | State parameter |

**Response:** Redirect with session cookie and access token.

---

### 3.3 Refresh Token

Refresh access token using session cookie.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/auth/refresh` |
| **Routes To** | Auth Service |
| **Auth Required** | Session cookie |

**Response:**
```json
{
  "success": true,
  "access_token": "eyJhbG...",
  "expires_in": 900
}
```

---

### 3.4 Logout

Terminate current session.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/auth/logout` |
| **Routes To** | Auth Service |
| **Auth Required** | Yes |

**Response:**
```json
{
  "success": true,
  "message": "Logged out"
}
```

---

### 3.5 WebSocket Token

Get one-time token for WebSocket authentication.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/auth/ws-token` |
| **Routes To** | Auth Service |
| **Auth Required** | Yes |

**Response:**
```json
{
  "success": true,
  "token": "ws_abc123...",
  "expires_in": 30
}
```

**Migration Note:** New endpoint. WebSocket connections now require a one-time token instead of passing access token directly.

---

## 4. Chat APIs

### 4.1 Send Message

Send a message to a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.postMessage` |
| **Routes To** | Command Service → NATS `MESSAGES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "channel": "ch_general",
  "text": "Hello, world!",
  "alias": "Custom Name",
  "emoji": ":robot:",
  "avatar": "https://example.com/avatar.png",
  "attachments": [],
  "thread_id": null
}
```

**Response:**
```json
{
  "success": true,
  "correlation_id": "corr_01HZ3K...",
  "message": {
    "message_id": "msg_01HZ3K...",
    "ts": "2026-02-01T10:30:00.123Z"
  }
}
```

**Migration Note:** Returns `202` instead of `200`. Message ID generated immediately but persistence is async.

---

### 4.2 Update Message

Edit an existing message.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.update` |
| **Routes To** | Command Service → NATS `MESSAGES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "channel": "ch_general",
  "message_id": "msg_01HZ3K...",
  "text": "Updated message text"
}
```

**Response:**
```json
{
  "success": true,
  "correlation_id": "corr_01HZ3K...",
  "message": {
    "message_id": "msg_01HZ3K...",
    "edited_at": "2026-02-01T10:35:00.123Z"
  }
}
```

---

### 4.3 Delete Message

Delete a message.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.delete` |
| **Routes To** | Command Service → NATS `MESSAGES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "channel": "ch_general",
  "message_id": "msg_01HZ3K..."
}
```

---

### 4.4 Get Message

Retrieve a single message by ID.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/chat.getMessage` |
| **Routes To** | Query Service → Cassandra |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `message_id` | string | Yes | Message ID |

**Response:**
```json
{
  "success": true,
  "message": {
    "message_id": "msg_01HZ3K...",
    "channel_id": "ch_general",
    "sender": {
      "user_id": "usr_alice",
      "username": "alice",
      "name": "Alice Smith"
    },
    "text": "Hello, world!",
    "ts": "2026-02-01T10:30:00.123Z",
    "edited_at": null,
    "attachments": [],
    "reactions": {}
  }
}
```

---

### 4.5 Pin Message

Pin a message to a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.pinMessage` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "message_id": "msg_01HZ3K..."
}
```

---

### 4.6 Unpin Message

Unpin a message from a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.unPinMessage` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

---

### 4.7 Star Message

Star a message for personal reference.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.starMessage` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

---

### 4.8 Get Pinned Messages

Get all pinned messages in a channel.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/chat.getPinnedMessages` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `channel_id` | string | Yes | Channel ID |

---

### 4.9 Get Starred Messages

Get user's starred messages.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/chat.getStarredMessages` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

## 5. Channel APIs

### 5.1 Create Channel

Create a new channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.create` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "name": "engineering",
  "description": "Engineering team discussions",
  "type": "public",
  "members": ["usr_alice", "usr_bob"],
  "read_only": false
}
```

**Response:**
```json
{
  "success": true,
  "correlation_id": "corr_01HZ3K...",
  "channel": {
    "channel_id": "ch_01HZ3K...",
    "name": "engineering"
  }
}
```

---

### 5.2 Get Channel Info

Get channel details.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/channels.info` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `channel_id` | string | Yes* | Channel ID |
| `channel_name` | string | Yes* | Channel name |

*One of `channel_id` or `channel_name` required.

**Response:**
```json
{
  "success": true,
  "channel": {
    "channel_id": "ch_general",
    "name": "general",
    "description": "General discussions",
    "type": "public",
    "created_at": "2026-01-01T00:00:00Z",
    "created_by": "usr_admin",
    "members_count": 150,
    "messages_count": 12500,
    "last_message_at": "2026-02-01T10:30:00Z"
  }
}
```

---

### 5.3 List Channels

List all public channels.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/channels.list` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | int | 0 | Pagination offset |
| `count` | int | 50 | Number of results |
| `sort` | string | `name` | Sort field |
| `query` | string | | Filter query (JSON) |

**Response:**
```json
{
  "success": true,
  "channels": [...],
  "count": 50,
  "offset": 0,
  "total": 234
}
```

---

### 5.4 List Joined Channels

List channels the current user has joined.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/channels.list.joined` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

### 5.5 Get Channel History

Get message history for a channel.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/channels.history` |
| **Routes To** | Query Service → Cassandra |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `channel_id` | string | | Required. Channel ID |
| `latest` | string | now | Latest timestamp |
| `oldest` | string | | Oldest timestamp |
| `count` | int | 50 | Number of messages |
| `inclusive` | bool | false | Include boundary messages |

**Response:**
```json
{
  "success": true,
  "messages": [
    {
      "message_id": "msg_01HZ3K...",
      "sender": { ... },
      "text": "Hello!",
      "ts": "2026-02-01T10:30:00.123Z"
    }
  ],
  "has_more": true
}
```

**Migration Note:** Now reads from Cassandra (was MongoDB). Cursor-based pagination using `message_id` is recommended over timestamp-based.

---

### 5.6 Get Channel Members

List members of a channel.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/channels.members` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `channel_id` | string | | Required |
| `offset` | int | 0 | Pagination offset |
| `count` | int | 50 | Number of results |

---

### 5.7 Join Channel

Join a public channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.join` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "channel_id": "ch_general"
}
```

---

### 5.8 Leave Channel

Leave a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.leave` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

---

### 5.9 Invite to Channel

Invite users to a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.invite` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "channel_id": "ch_engineering",
  "user_id": "usr_carol"
}
```

---

### 5.10 Kick from Channel

Remove a user from a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.kick` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes (moderator) |
| **Response** | `202 Accepted` |

---

### 5.11 Set Channel Topic

Set channel topic/description.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.setTopic` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "channel_id": "ch_general",
  "topic": "Welcome to the general channel!"
}
```

---

### 5.12 Archive Channel

Archive a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.archive` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes (admin) |
| **Response** | `202 Accepted` |

---

### 5.13 Delete Channel

Permanently delete a channel.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/channels.delete` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes (admin) |
| **Response** | `202 Accepted` |

---

## 6. Direct Message APIs

### 6.1 Create DM

Create or get existing DM with a user.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/dm.create` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |

**Request Body:**
```json
{
  "username": "bob"
}
```

**Response:**
```json
{
  "success": true,
  "room": {
    "room_id": "dm_01HZ3K...",
    "type": "dm",
    "usernames": ["alice", "bob"]
  }
}
```

---

### 6.2 List DMs

List user's direct message conversations.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/dm.list` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

### 6.3 Get DM History

Get message history for a DM.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/dm.history` |
| **Routes To** | Query Service → Cassandra |
| **Auth Required** | Yes |

**Query Parameters:** Same as `channels.history`

---

### 6.4 Send DM

Send a direct message. Uses same endpoint as channel messages.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.postMessage` |
| **Routes To** | Command Service |
| **Auth Required** | Yes |

**Request Body:**
```json
{
  "channel": "dm_01HZ3K...",
  "text": "Hey, got a minute?"
}
```

---

## 7. User APIs

### 7.1 Get Current User

Get current authenticated user's info.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/me` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Response:**
```json
{
  "success": true,
  "user_id": "usr_01HZ3K...",
  "username": "alice",
  "name": "Alice Smith",
  "email": "alice@example.com",
  "status": "online",
  "avatar_url": "https://...",
  "roles": ["user"],
  "preferences": { ... }
}
```

---

### 7.2 Get User Info

Get another user's public info.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/users.info` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `user_id` | string | Yes* | User ID |
| `username` | string | Yes* | Username |

---

### 7.3 List Users

List all users (with pagination).

| | |
|---|---|
| **Endpoint** | `GET /api/v1/users.list` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

### 7.4 Update Profile

Update current user's profile.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/users.updateOwnBasicInfo` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "name": "Alice Smith-Jones",
  "username": "alicej",
  "status_text": "Working from home"
}
```

**Migration Note:** Profile fields synced from Keycloak (email, name) cannot be changed here.

---

### 7.5 Set Status

Set user's presence status.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/users.setStatus` |
| **Routes To** | Command Service → Redis (presence) |
| **Auth Required** | Yes |

**Request Body:**
```json
{
  "status": "away",
  "message": "In a meeting"
}
```

**Status values:** `online`, `away`, `busy`, `offline`

---

### 7.6 Get User Presence

Get a user's presence status.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/users.getPresence` |
| **Routes To** | Query Service → Redis |
| **Auth Required** | Yes |

---

### 7.7 Set Avatar

Update user's avatar.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/users.setAvatar` |
| **Routes To** | Command Service |
| **Auth Required** | Yes |
| **Content-Type** | `multipart/form-data` |

---

### 7.8 Update Preferences

Update user preferences (notifications, theme, etc.).

| | |
|---|---|
| **Endpoint** | `POST /api/v1/users.setPreferences` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "preferences": {
    "language": "en",
    "theme": "dark",
    "notifications": {
      "desktop": true,
      "mobile": true
    }
  }
}
```

---

## 8. Team APIs

### 8.1 Create Team

Create a new team (group of channels).

| | |
|---|---|
| **Endpoint** | `POST /api/v1/teams.create` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "name": "Engineering",
  "type": "private",
  "members": ["usr_alice", "usr_bob"]
}
```

---

### 8.2 List Teams

List teams the user belongs to.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/teams.list` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

### 8.3 Get Team Info

Get team details.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/teams.info` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

### 8.4 Add Team Members

Add members to a team.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/teams.addMembers` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes (team admin) |
| **Response** | `202 Accepted` |

---

### 8.5 List Team Channels

List channels belonging to a team.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/teams.listChannels` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

## 9. Thread APIs

### 9.1 Get Thread Messages

Get messages in a thread.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/chat.getThreadMessages` |
| **Routes To** | Query Service → Cassandra |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `thread_id` | string | Yes | Parent message ID |
| `count` | int | No | Number of messages (default 50) |
| `offset` | int | No | Pagination offset |

**Response:**
```json
{
  "success": true,
  "messages": [...],
  "thread_info": {
    "thread_id": "msg_01HZ3K...",
    "reply_count": 15,
    "participants": ["usr_alice", "usr_bob", "usr_carol"],
    "last_reply_at": "2026-02-01T10:30:00Z"
  }
}
```

---

### 9.2 Reply to Thread

Send a reply to a thread.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.postMessage` |
| **Routes To** | Command Service → NATS `MESSAGES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "channel": "ch_general",
  "thread_id": "msg_01HZ3K...",
  "text": "This is a thread reply"
}
```

---

### 9.3 Follow Thread

Follow a thread to receive notifications.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.followMessage` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "message_id": "msg_01HZ3K..."
}
```

---

### 9.4 Unfollow Thread

Stop following a thread.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.unfollowMessage` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

---

### 9.5 Get Threads List

Get aggregated list of threads from all channels.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/chat.getThreadsList` |
| **Routes To** | Query Service → Cassandra |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `type` | string | `following` | `following`, `unread`, `all` |
| `count` | int | 50 | Number of threads |
| `offset` | int | 0 | Pagination offset |

**Response:**
```json
{
  "success": true,
  "threads": [
    {
      "thread_id": "msg_01HZ3K...",
      "channel_id": "ch_general",
      "channel_name": "general",
      "parent_message": {
        "text": "Original message...",
        "sender": { ... }
      },
      "reply_count": 15,
      "unread_count": 3,
      "last_reply": {
        "text": "Latest reply...",
        "sender": { ... },
        "ts": "2026-02-01T10:30:00Z"
      },
      "participants": [...]
    }
  ]
}
```

**Migration Note:** New endpoint for Threads View feature.

---

## 10. File APIs

### 10.1 Get Upload URL

Get presigned URL for direct S3 upload.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/files.getUploadUrl` |
| **Routes To** | Command Service |
| **Auth Required** | Yes |

**Request Body:**
```json
{
  "filename": "screenshot.png",
  "content_type": "image/png",
  "size": 245678,
  "channel_id": "ch_general"
}
```

**Response:**
```json
{
  "success": true,
  "file_id": "file_01HZ3K...",
  "upload_url": "https://s3.example.com/uploads/...",
  "upload_fields": {
    "key": "uploads/file_01HZ3K...",
    "policy": "...",
    "x-amz-signature": "..."
  },
  "expires_at": "2026-02-01T10:35:00Z"
}
```

**Migration Note:** New direct-to-S3 upload pattern. Replaces multipart upload to application server.

---

### 10.2 Complete Upload

Mark upload as complete and trigger processing.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/files.completeUpload` |
| **Routes To** | Command Service → NATS `FILES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "file_id": "file_01HZ3K...",
  "channel_id": "ch_general",
  "message_text": "Check out this screenshot"
}
```

---

### 10.3 Get File Info

Get file metadata and download URLs.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/files.info` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_id` | string | Yes | File ID |

**Response:**
```json
{
  "success": true,
  "file": {
    "file_id": "file_01HZ3K...",
    "filename": "screenshot.png",
    "content_type": "image/png",
    "size": 245678,
    "download_url": "https://cdn.example.com/files/...",
    "thumbnail_url": "https://cdn.example.com/thumbs/...",
    "uploaded_by": "usr_alice",
    "uploaded_at": "2026-02-01T10:30:00Z",
    "status": "ready"
  }
}
```

---

### 10.4 List Files

List files in a channel.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/files.list` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

### 10.5 Delete File

Delete an uploaded file.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/files.delete` |
| **Routes To** | Command Service → NATS `FILES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

---

## 11. Reaction APIs

### 11.1 Add Reaction

Add a reaction to a message.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.react` |
| **Routes To** | Command Service → NATS `MESSAGES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "message_id": "msg_01HZ3K...",
  "emoji": "thumbsup",
  "should_react": true
}
```

---

### 11.2 Remove Reaction

Remove a reaction from a message.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/chat.react` |
| **Routes To** | Command Service → NATS `MESSAGES` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

**Request Body:**
```json
{
  "message_id": "msg_01HZ3K...",
  "emoji": "thumbsup",
  "should_react": false
}
```

---

### 11.3 Get Message Reactions

Get all reactions on a message.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/chat.getMessageReactions` |
| **Routes To** | Query Service → Cassandra |
| **Auth Required** | Yes |

---

## 12. Search APIs

### 12.1 Search Messages

Full-text search across messages.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/chat.search` |
| **Routes To** | Query Service → Elasticsearch |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` | string | | Required. Search query |
| `channel_id` | string | | Filter by channel |
| `from_user` | string | | Filter by sender |
| `from_date` | string | | Start date (ISO 8601) |
| `to_date` | string | | End date (ISO 8601) |
| `count` | int | 20 | Number of results |
| `offset` | int | 0 | Pagination offset |

**Response:**
```json
{
  "success": true,
  "messages": [
    {
      "message_id": "msg_01HZ3K...",
      "channel_id": "ch_general",
      "channel_name": "general",
      "text": "... <em>search term</em> ...",
      "sender": { ... },
      "ts": "2026-02-01T10:30:00Z",
      "score": 0.95
    }
  ],
  "total": 42
}
```

**Migration Note:** Now powered by Elasticsearch instead of MongoDB text search.

---

### 12.2 Search Users

Search for users by name/username.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/users.autocomplete` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `selector` | string | Search term |

---

### 12.3 Search Channels

Search for channels by name.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/channels.autocomplete` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

## 13. Push APIs

### 13.1 Register Device

Register device for push notifications.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/push/devices` |
| **Routes To** | Command Service → MongoDB |
| **Auth Required** | Yes |

**Request Body:**
```json
{
  "platform": "ios",
  "token": "abc123...",
  "device_id": "device_01HZ3K...",
  "app_version": "3.2.1"
}
```

---

### 13.2 Unregister Device

Remove device from push notifications.

| | |
|---|---|
| **Endpoint** | `DELETE /api/v1/push/devices/{device_id}` |
| **Routes To** | Command Service → MongoDB |
| **Auth Required** | Yes |

---

### 13.3 Get Push Preferences

Get user's push notification preferences.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/push/preferences` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes |

---

### 13.4 Update Push Preferences

Update push notification preferences.

| | |
|---|---|
| **Endpoint** | `PUT /api/v1/push/preferences` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes |
| **Response** | `202 Accepted` |

---

## 14. Bot APIs

### 14.1 Create Bot

Register a new bot application.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/bots` |
| **Routes To** | Command Service → MongoDB |
| **Auth Required** | Yes (admin) |

See [Bot Integration](../features/bot-integration.md) for details.

---

### 14.2 List Bots

List registered bots.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/bots` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes (admin) |

---

### 14.3 Incoming Webhook

Post message via incoming webhook (bot authentication).

| | |
|---|---|
| **Endpoint** | `POST /hooks/{token}` |
| **Routes To** | Command Service → NATS `MESSAGES` |
| **Auth Required** | Webhook token |

---

## 15. Admin APIs

### 15.1 Get Server Info

Get server status and statistics.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/info` |
| **Routes To** | Query Service |
| **Auth Required** | Yes (admin) |

---

### 15.2 Get Statistics

Get platform usage statistics.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/statistics` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes (admin) |

---

### 15.3 List All Users (Admin)

List all users with admin details.

| | |
|---|---|
| **Endpoint** | `GET /api/v1/users.list` |
| **Routes To** | Query Service → MongoDB |
| **Auth Required** | Yes (admin) |

---

### 15.4 Deactivate User

Deactivate a user account.

| | |
|---|---|
| **Endpoint** | `POST /api/v1/users.setActiveStatus` |
| **Routes To** | Command Service → NATS `META` |
| **Auth Required** | Yes (admin) |
| **Response** | `202 Accepted` |

---

## 16. WebSocket Events

### 16.1 Connection

Connect to WebSocket endpoint with one-time token:

```
wss://ws.example.com/websocket?token={ws_token}
```

### 16.2 Server → Client Events

| Event Type | Description |
|------------|-------------|
| `auth.ok` | Authentication successful |
| `message.new` | New message in channel |
| `message.edited` | Message was edited |
| `message.deleted` | Message was deleted |
| `typing` | User typing indicator |
| `presence` | User presence changed |
| `member.joined` | User joined channel |
| `member.left` | User left channel |
| `thread.reply` | New thread reply |
| `reaction.added` | Reaction added |
| `reaction.removed` | Reaction removed |
| `notification` | Push notification |

### 16.3 Client → Server Events

| Event Type | Description |
|------------|-------------|
| `ping` | Keepalive ping |
| `typing.start` | Start typing indicator |
| `typing.stop` | Stop typing indicator |
| `mark_read` | Mark channel as read |
| `channel.focus` | User focused on channel |
| `channel.blur` | User left channel view |

See [Notification Service](../detailed-design.md#15-notification-service) for protocol details.

---

## 17. Migration Notes

### 17.1 Breaking Changes

| Change | Old Behavior | New Behavior |
|--------|--------------|--------------|
| **Authentication** | Username/password login | Keycloak OIDC only |
| **Write responses** | Synchronous `200 OK` | Async `202 Accepted` |
| **WebSocket auth** | Access token in URL | One-time WS token |
| **File uploads** | Multipart to app server | Direct S3 presigned URL |
| **DDP protocol** | Meteor DDP | JSON over WebSocket |

### 17.2 Deprecated Endpoints

| Endpoint | Status | Alternative |
|----------|--------|-------------|
| `POST /api/v1/login` | Removed | `GET /api/v1/auth/login` |
| `POST /api/v1/logout` | Changed | `POST /api/v1/auth/logout` |
| `GET /api/v1/me` | Unchanged | — |
| `POST /api/v1/rooms.upload` | Deprecated | `POST /api/v1/files.getUploadUrl` |

### 17.3 Response Code Changes

All write operations now return:
- `202 Accepted` — Command accepted, processing async
- `correlation_id` — For tracking request through system

Clients should:
1. Check for WebSocket event confirmation
2. Use `correlation_id` for debugging
3. Handle eventual consistency (brief read-after-write delay)

### 17.4 Rate Limits

| Endpoint Category | Limit | Period |
|-------------------|-------|--------|
| Auth endpoints | 10 | 1 min |
| Read endpoints | 100 | 1 min |
| Write endpoints | 60 | 1 min |
| Search | 30 | 1 min |
| File upload | 20 | 1 min |

---

## Related Documents

- [Detailed Design](../detailed-design.md) — Service architecture
- [Authentication](../features/authentication.md) — OIDC login flow
- [Bot Integration](../features/bot-integration.md) — Bot APIs
- [File Uploads](../features/file-uploads.md) — S3 upload flow
