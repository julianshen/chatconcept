# Bot Integration

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [Bot Types](#2-bot-types)
3. [Bot Registration](#3-bot-registration)
4. [Bot Authentication](#4-bot-authentication)
5. [Outgoing Webhooks](#5-outgoing-webhooks)
6. [Incoming Webhooks](#6-incoming-webhooks)
7. [Bot REST API](#7-bot-rest-api)
8. [Slash Commands](#8-slash-commands)
9. [Interactive Messages](#9-interactive-messages)
10. [Bot Permissions](#10-bot-permissions)
11. [Rate Limiting](#11-rate-limiting)
12. [Bot Development Guide](#12-bot-development-guide)

---

## 1. Overview

The platform supports bot integration for automation, external service connections, and custom workflows. Bots can interact with the platform through multiple mechanisms depending on their requirements.

### Integration Patterns

| Pattern | Direction | Use Case |
|---------|-----------|----------|
| **Outgoing Webhooks** | Platform → Bot | React to events (messages, joins, reactions) |
| **Incoming Webhooks** | Bot → Platform | Post messages, notifications, alerts |
| **REST API** | Bot ↔ Platform | Full bidirectional interaction |
| **Slash Commands** | User → Bot | User-triggered bot actions |
| **Interactive Messages** | User ↔ Bot | Button clicks, form submissions |

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Bot Integration Architecture                        │
│                                                                              │
│  ┌─────────────────┐                                                         │
│  │  Bot Server     │   (Developer-hosted)                                    │
│  │                 │                                                         │
│  │  ┌───────────┐  │                                                         │
│  │  │ Webhook   │◄─┼──────────── Outgoing Webhooks (events)                  │
│  │  │ Handler   │  │                                                         │
│  │  └───────────┘  │                                                         │
│  │                 │                                                         │
│  │  ┌───────────┐  │                                                         │
│  │  │ API       │──┼──────────── REST API / Incoming Webhooks                │
│  │  │ Client    │  │                                                         │
│  │  └───────────┘  │                                                         │
│  └─────────────────┘                                                         │
│           │                                                                  │
│           │ HTTPS                                                            │
│           ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                         API Gateway                                      ││
│  │  • Bot token validation                                                  ││
│  │  • Rate limiting                                                         ││
│  │  • Scope enforcement                                                     ││
│  └────────────────────────────────┬────────────────────────────────────────┘│
│                                   │                                          │
│       ┌───────────────────────────┼───────────────────────────┐             │
│       │                           │                           │             │
│       ▼                           ▼                           ▼             │
│  ┌──────────┐              ┌──────────┐              ┌──────────────┐       │
│  │ Command  │              │  Query   │              │   Webhook    │       │
│  │ Service  │              │ Service  │              │  Dispatcher  │       │
│  └──────────┘              └──────────┘              └──────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bot Types

### 2.1 Webhook Bots

Simple bots that receive events via HTTP webhooks and respond via incoming webhooks.

**Characteristics:**
- Stateless, event-driven
- No persistent connection required
- Easy to deploy (any HTTP server)
- Limited to webhook-triggered interactions

**Best for:** Notifications, simple automations, CI/CD integrations

### 2.2 API Bots

Full-featured bots that authenticate with bot tokens and use the REST API.

**Characteristics:**
- Can initiate actions (not just respond)
- Access to full API capabilities
- Can maintain state
- Require bot token management

**Best for:** Complex workflows, interactive bots, administrative tasks

### 2.3 Hybrid Bots

Combine webhooks for real-time events with API access for proactive actions.

**Best for:** Most production bots

---

## 3. Bot Registration

### 3.1 Create Bot Application

Bots are registered through the admin interface or API:

```
POST /api/v1/bots
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "name": "CI/CD Bot",
  "description": "Reports build and deployment status",
  "icon_url": "https://example.com/bot-icon.png",
  "owner_id": "usr_01HZ3K...",
  "scopes": [
    "messages:write",
    "channels:read",
    "users:read"
  ],
  "webhook_url": "https://bot.example.com/webhook",
  "slash_commands": [
    {
      "command": "deploy",
      "description": "Trigger deployment",
      "usage": "/deploy [environment] [version]"
    }
  ]
}
```

### 3.2 Bot Application Response

```json
{
  "bot_id": "bot_01HZ3K4M5N",
  "app_id": "app_01HZ3K4M5N",
  "name": "CI/CD Bot",
  "bot_user_id": "usr_bot_01HZ3K...",
  "credentials": {
    "bot_token": "xbot-01HZ3K4M5N6P7Q8R9S0T...",
    "webhook_secret": "whsec_abc123...",
    "incoming_webhook_url": "https://chat.example.com/hooks/inc_01HZ3K..."
  },
  "status": "active",
  "created_at": "2026-02-01T10:30:00Z"
}
```

### 3.3 MongoDB: Bot Schema

```javascript
// Collection: bots
{
  _id: ObjectId,
  bot_id: "bot_01HZ3K4M5N",
  app_id: "app_01HZ3K4M5N",

  // Bot identity
  name: "CI/CD Bot",
  description: "Reports build and deployment status",
  icon_url: "https://example.com/bot-icon.png",

  // Associated bot user (for sending messages)
  bot_user_id: "usr_bot_01HZ3K...",

  // Owner
  owner_id: "usr_01HZ3K...",
  owner_type: "user",  // "user" | "team" | "organization"

  // Credentials (hashed)
  bot_token_hash: "sha256:...",
  webhook_secret_hash: "sha256:...",
  incoming_webhook_token: "inc_01HZ3K...",

  // Configuration
  scopes: ["messages:write", "channels:read", "users:read"],
  webhook_url: "https://bot.example.com/webhook",
  webhook_events: ["message.sent", "message.reaction", "member.joined"],

  // Slash commands
  slash_commands: [
    {
      command: "deploy",
      description: "Trigger deployment",
      usage: "/deploy [environment] [version]",
      channels: []  // empty = all channels where bot is installed
    }
  ],

  // Status
  status: "active",  // "active" | "disabled" | "suspended"

  // Installation tracking
  installed_channels: ["ch_general", "ch_engineering"],
  installed_at: ISODate("2026-02-01T10:30:00Z"),

  // Timestamps
  created_at: ISODate("2026-02-01T10:30:00Z"),
  updated_at: ISODate("2026-02-01T10:30:00Z")
}

// Indexes
{ bot_id: 1 }                    // unique
{ app_id: 1 }                    // unique
{ bot_user_id: 1 }               // unique
{ owner_id: 1 }
{ "slash_commands.command": 1 }
{ installed_channels: 1 }
```

---

## 4. Bot Authentication

### 4.1 Bot Tokens

Bots authenticate using long-lived bot tokens (not OIDC):

```
Authorization: Bearer xbot-01HZ3K4M5N6P7Q8R9S0T...
```

**Token format:** `xbot-{bot_id}{random_bytes}`

**Token properties:**
- Long-lived (no expiration by default)
- Can be rotated manually
- Scoped to bot's permissions
- Stored hashed in database

### 4.2 Token Validation (API Gateway)

```yaml
# envoy.yaml - Bot token validation
http_filters:
- name: envoy.filters.http.ext_authz
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
    http_service:
      server_uri:
        uri: "http://auth-service:8080/auth/validate-bot"
        cluster: auth_service
        timeout: 0.5s
      authorization_request:
        headers_to_add:
          - key: "x-original-uri"
            value: "%REQ(:path)%"
      authorization_response:
        allowed_upstream_headers:
          patterns:
            - exact: "x-bot-id"
            - exact: "x-bot-scopes"
```

### 4.3 Auth Service: Bot Token Validation

```go
func (s *AuthService) ValidateBotToken(ctx context.Context, token string) (*BotClaims, error) {
    // 1. Extract bot ID from token prefix
    if !strings.HasPrefix(token, "xbot-") {
        return nil, fmt.Errorf("invalid bot token format")
    }

    botID := extractBotID(token)

    // 2. Get bot from database
    bot, err := s.mongo.FindOne(ctx, "bots", bson.M{"bot_id": botID})
    if err != nil {
        return nil, fmt.Errorf("bot not found")
    }

    // 3. Verify token hash
    tokenHash := sha256Hash(token)
    if !secureCompare(tokenHash, bot.BotTokenHash) {
        return nil, fmt.Errorf("invalid bot token")
    }

    // 4. Check bot status
    if bot.Status != "active" {
        return nil, fmt.Errorf("bot is %s", bot.Status)
    }

    return &BotClaims{
        BotID:     bot.BotID,
        BotUserID: bot.BotUserID,
        Scopes:    bot.Scopes,
        Channels:  bot.InstalledChannels,
    }, nil
}
```

### 4.4 Token Rotation

```
POST /api/v1/bots/{bot_id}/rotate-token
Authorization: Bearer <admin_token>

Response:
{
  "bot_token": "xbot-01HZ3K4M5N-NEW-TOKEN...",
  "previous_token_valid_until": "2026-02-01T11:30:00Z"
}
```

Previous token remains valid for 1 hour to allow graceful migration.

---

## 5. Outgoing Webhooks

Outgoing webhooks deliver platform events to bot servers.

### 5.1 Event Types

| Event | Trigger | Payload |
|-------|---------|---------|
| `message.sent` | New message in channel | Message content, sender, channel |
| `message.edited` | Message edited | Updated message, editor |
| `message.deleted` | Message deleted | Message ID, deleter |
| `message.reaction` | Reaction added/removed | Message ID, reaction, user |
| `member.joined` | User joins channel | User, channel |
| `member.left` | User leaves channel | User, channel |
| `channel.created` | New channel created | Channel details |
| `channel.updated` | Channel settings changed | Updated channel |
| `slash_command` | User invokes slash command | Command, args, user, channel |
| `interactive.action` | Button click / form submit | Action ID, user, context |

### 5.2 Webhook Payload Format

```json
{
  "webhook_id": "whk_01HZ3K...",
  "event_id": "evt_01HZ3K4M5N6P7Q8R9S0T",
  "event_type": "message.sent",
  "timestamp": "2026-02-01T10:30:00.123Z",
  "bot_id": "bot_01HZ3K4M5N",

  "channel": {
    "channel_id": "ch_general",
    "name": "general",
    "type": "public"
  },

  "sender": {
    "user_id": "usr_bob",
    "username": "bob",
    "display_name": "Bob Smith",
    "is_bot": false
  },

  "payload": {
    "message_id": "msg_01HZ3K...",
    "text": "Hello @cicd-bot /deploy production v1.2.3",
    "mentions": [
      {"user_id": "usr_bot_01HZ3K...", "type": "bot"}
    ],
    "attachments": [],
    "thread_id": null
  }
}
```

### 5.3 Webhook HTTP Headers

```http
POST /webhook HTTP/1.1
Host: bot.example.com
Content-Type: application/json
X-Webhook-ID: whk_01HZ3K...
X-Webhook-Signature: sha256=d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592
X-Webhook-Timestamp: 2026-02-01T10:30:00.123Z
X-Event-Type: message.sent
X-Bot-ID: bot_01HZ3K4M5N
X-Request-ID: req_abc123
User-Agent: ChatPlatform-Webhook/1.0
```

### 5.4 Signature Verification

```go
// Bot server: Verify webhook signature
func verifyWebhookSignature(payload []byte, secret, signature, timestamp string) bool {
    // 1. Check timestamp freshness (prevent replay attacks)
    ts, _ := time.Parse(time.RFC3339, timestamp)
    if time.Since(ts) > 5*time.Minute {
        return false
    }

    // 2. Compute expected signature
    signedPayload := fmt.Sprintf("%s.%s", timestamp, string(payload))
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write([]byte(signedPayload))
    expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))

    // 3. Compare signatures
    return hmac.Equal([]byte(expected), []byte(signature))
}
```

### 5.5 Webhook Delivery Flow

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐     ┌────────────┐
│  Event   │     │   WEBHOOKS   │     │  Webhook    │     │    Bot     │
│ (NATS)   │────►│   Stream     │────►│ Dispatcher  │────►│   Server   │
└──────────┘     └──────────────┘     └──────┬──────┘     └────────────┘
                                             │
                                             │ On failure:
                                             │ Retry with backoff
                                             │ (5s, 30s, 2m, 15m, 1h)
                                             │
                                             │ After 5 failures:
                                             │ Move to DLQ
                                             ▼
                                      ┌─────────────┐
                                      │  WEBHOOKS   │
                                      │     DLQ     │
                                      └─────────────┘
```

### 5.6 Webhook Response Handling

Bots can respond directly to webhooks for immediate actions:

```json
// Bot server response (optional)
{
  "response_type": "in_channel",  // "in_channel" | "ephemeral"
  "text": "Deployment started for v1.2.3 to production",
  "attachments": [
    {
      "type": "adaptive_card",
      "card": { ... }
    }
  ]
}
```

---

## 6. Incoming Webhooks

Incoming webhooks allow bots to post messages without full API authentication.

### 6.1 Incoming Webhook URL

Each bot receives a unique incoming webhook URL:

```
https://chat.example.com/hooks/inc_01HZ3K4M5N6P7Q8R9S0T
```

### 6.2 Post Message via Incoming Webhook

```
POST /hooks/inc_01HZ3K4M5N6P7Q8R9S0T
Content-Type: application/json

{
  "channel_id": "ch_general",
  "text": "Build #1234 completed successfully! :white_check_mark:",
  "username": "CI/CD Bot",
  "icon_emoji": ":robot:",
  "attachments": [
    {
      "color": "#36a64f",
      "title": "Build Details",
      "title_link": "https://ci.example.com/builds/1234",
      "fields": [
        {"title": "Status", "value": "Success", "short": true},
        {"title": "Duration", "value": "3m 42s", "short": true},
        {"title": "Commit", "value": "abc1234", "short": true},
        {"title": "Branch", "value": "main", "short": true}
      ]
    }
  ]
}
```

### 6.3 Incoming Webhook Processing

```go
func (h *WebhookHandler) HandleIncoming(w http.ResponseWriter, r *http.Request) {
    // 1. Extract webhook token from URL
    token := chi.URLParam(r, "token")

    // 2. Validate token and get bot
    bot, err := h.validateIncomingWebhook(r.Context(), token)
    if err != nil {
        http.Error(w, "invalid webhook", http.StatusUnauthorized)
        return
    }

    // 3. Parse payload
    var payload IncomingWebhookPayload
    json.NewDecoder(r.Body).Decode(&payload)

    // 4. Validate channel access
    if !contains(bot.InstalledChannels, payload.ChannelID) {
        http.Error(w, "bot not installed in channel", http.StatusForbidden)
        return
    }

    // 5. Publish message event
    event := MessageSentEvent{
        MessageID: generateMessageID(),
        ChannelID: payload.ChannelID,
        SenderID:  bot.BotUserID,
        Text:      payload.Text,
        Attachments: payload.Attachments,
        IsBot:     true,
    }
    h.nats.Publish("messages.send."+payload.ChannelID, event)

    w.WriteHeader(http.StatusAccepted)
}
```

### 6.4 Posting to Threads

```json
{
  "channel_id": "ch_general",
  "thread_id": "msg_01HZ3K...",
  "text": "Deployment to production complete!"
}
```

---

## 7. Bot REST API

Bots with API tokens can access the full REST API.

### 7.1 API Endpoints for Bots

| Endpoint | Method | Scope Required | Description |
|----------|--------|----------------|-------------|
| `/api/v1/chat.postMessage` | POST | `messages:write` | Send message |
| `/api/v1/chat.update` | POST | `messages:write` | Edit message |
| `/api/v1/chat.delete` | POST | `messages:write` | Delete message |
| `/api/v1/channels.list` | GET | `channels:read` | List channels |
| `/api/v1/channels.info` | GET | `channels:read` | Get channel info |
| `/api/v1/channels.history` | GET | `channels:history` | Get message history |
| `/api/v1/users.info` | GET | `users:read` | Get user info |
| `/api/v1/users.list` | GET | `users:read` | List users |
| `/api/v1/reactions.add` | POST | `reactions:write` | Add reaction |
| `/api/v1/files.upload` | POST | `files:write` | Upload file |

### 7.2 Send Message via API

```
POST /api/v1/chat.postMessage
Authorization: Bearer xbot-01HZ3K4M5N...
Content-Type: application/json

{
  "channel_id": "ch_general",
  "text": "Hello from the bot!",
  "attachments": [
    {
      "type": "adaptive_card",
      "card": {
        "type": "AdaptiveCard",
        "version": "1.5",
        "body": [
          {
            "type": "TextBlock",
            "text": "Action Required",
            "weight": "bolder",
            "size": "medium"
          }
        ],
        "actions": [
          {
            "type": "Action.Submit",
            "title": "Approve",
            "data": {"action": "approve", "request_id": "req_123"}
          },
          {
            "type": "Action.Submit",
            "title": "Reject",
            "data": {"action": "reject", "request_id": "req_123"}
          }
        ]
      }
    }
  ]
}
```

### 7.3 Response

```json
{
  "ok": true,
  "message_id": "msg_01HZ3K...",
  "channel_id": "ch_general",
  "ts": "2026-02-01T10:30:00.123Z"
}
```

---

## 8. Slash Commands

### 8.1 Command Registration

Slash commands are defined during bot registration:

```json
{
  "slash_commands": [
    {
      "command": "deploy",
      "description": "Trigger a deployment",
      "usage": "/deploy <environment> <version>",
      "options": [
        {
          "name": "environment",
          "type": "string",
          "required": true,
          "choices": ["staging", "production"]
        },
        {
          "name": "version",
          "type": "string",
          "required": true
        }
      ]
    },
    {
      "command": "status",
      "description": "Check deployment status",
      "usage": "/status [environment]"
    }
  ]
}
```

### 8.2 Command Invocation Flow

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐     ┌────────────┐
│   User   │     │   Command    │     │  Webhook    │     │    Bot     │
│  types   │────►│   Service    │────►│ Dispatcher  │────►│   Server   │
│ /deploy  │     │              │     │             │     │            │
└──────────┘     └──────────────┘     └─────────────┘     └─────┬──────┘
                                                                │
                                                                │ Process
                                                                │ command
                                                                ▼
                                                          ┌────────────┐
                                                          │  Response  │
                                                          │  (webhook  │
                                                          │  or API)   │
                                                          └────────────┘
```

### 8.3 Slash Command Webhook Payload

```json
{
  "event_type": "slash_command",
  "command": "deploy",
  "text": "production v1.2.3",
  "args": {
    "environment": "production",
    "version": "v1.2.3"
  },
  "user": {
    "user_id": "usr_alice",
    "username": "alice"
  },
  "channel": {
    "channel_id": "ch_engineering",
    "name": "engineering"
  },
  "response_url": "https://chat.example.com/hooks/response/resp_01HZ3K...",
  "trigger_id": "trig_01HZ3K..."
}
```

### 8.4 Command Response Options

**Immediate response (webhook return):**
```json
{
  "response_type": "in_channel",
  "text": "Starting deployment..."
}
```

**Delayed response (via response_url):**
```
POST https://chat.example.com/hooks/response/resp_01HZ3K...
Content-Type: application/json

{
  "response_type": "in_channel",
  "text": "Deployment to production completed!",
  "replace_original": false
}
```

---

## 9. Interactive Messages

### 9.1 Adaptive Card Actions

When users interact with adaptive card actions, the bot receives a webhook:

```json
{
  "event_type": "interactive.action",
  "action": {
    "action_id": "approve_button",
    "action_type": "Action.Submit",
    "data": {
      "action": "approve",
      "request_id": "req_123"
    }
  },
  "user": {
    "user_id": "usr_alice",
    "username": "alice"
  },
  "channel": {
    "channel_id": "ch_engineering"
  },
  "message": {
    "message_id": "msg_01HZ3K...",
    "text": "Approval Request"
  },
  "response_url": "https://chat.example.com/hooks/response/resp_01HZ3K..."
}
```

### 9.2 Update Original Message

Bots can update the original message after an action:

```
POST https://chat.example.com/hooks/response/resp_01HZ3K...
Content-Type: application/json

{
  "replace_original": true,
  "text": "Request approved by @alice",
  "attachments": [
    {
      "type": "adaptive_card",
      "card": {
        "type": "AdaptiveCard",
        "body": [
          {
            "type": "TextBlock",
            "text": "✅ Approved by alice",
            "color": "good"
          }
        ]
      }
    }
  ]
}
```

### 9.3 Modal Dialogs

Bots can open modal dialogs for complex input:

```json
// Triggered by slash command or button
{
  "trigger_id": "trig_01HZ3K...",
  "modal": {
    "type": "modal",
    "title": "Deploy Configuration",
    "submit_label": "Deploy",
    "close_label": "Cancel",
    "callback_id": "deploy_modal",
    "elements": [
      {
        "type": "select",
        "name": "environment",
        "label": "Environment",
        "options": [
          {"label": "Staging", "value": "staging"},
          {"label": "Production", "value": "production"}
        ]
      },
      {
        "type": "text_input",
        "name": "version",
        "label": "Version",
        "placeholder": "v1.2.3"
      },
      {
        "type": "checkbox",
        "name": "notify",
        "label": "Notify channel on completion",
        "initial_value": true
      }
    ]
  }
}
```

---

## 10. Bot Permissions

### 10.1 Scopes

| Scope | Access |
|-------|--------|
| `messages:read` | Read messages in installed channels |
| `messages:write` | Send messages, reply to threads |
| `messages:history` | Access message history |
| `channels:read` | List channels, get channel info |
| `channels:write` | Create channels, update settings |
| `channels:manage` | Archive, delete channels |
| `users:read` | List users, get user profiles |
| `reactions:write` | Add/remove reactions |
| `files:read` | Access file metadata |
| `files:write` | Upload files |
| `slash_commands` | Respond to slash commands |
| `interactive` | Receive interactive message events |

### 10.2 Channel Installation

Bots must be installed in channels to access them:

```
POST /api/v1/bots/{bot_id}/install
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "channel_id": "ch_engineering",
  "permissions": ["messages:read", "messages:write"]
}
```

### 10.3 Permission Checks

```go
func (h *Handler) PostMessage(w http.ResponseWriter, r *http.Request) {
    botClaims := r.Context().Value("bot_claims").(*BotClaims)

    // Check scope
    if !hasScope(botClaims.Scopes, "messages:write") {
        http.Error(w, "missing scope: messages:write", http.StatusForbidden)
        return
    }

    // Check channel installation
    if !contains(botClaims.Channels, req.ChannelID) {
        http.Error(w, "bot not installed in channel", http.StatusForbidden)
        return
    }

    // Process request...
}
```

---

## 11. Rate Limiting

### 11.1 Rate Limits by Tier

| Tier | Messages/min | API calls/min | Webhooks/min |
|------|--------------|---------------|--------------|
| **Free** | 60 | 100 | 30 |
| **Standard** | 300 | 500 | 100 |
| **Enterprise** | 1000 | 2000 | 500 |

### 11.2 Rate Limit Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 245
X-RateLimit-Reset: 1706789160
```

### 11.3 Rate Limit Exceeded

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{
  "ok": false,
  "error": "rate_limited",
  "retry_after": 30
}
```

### 11.4 Redis Rate Limiting

```
Key:    ratelimit:bot:{bot_id}:messages
Type:   String (counter)
Value:  count
TTL:    60 seconds

Key:    ratelimit:bot:{bot_id}:api
Type:   String (counter)
Value:  count
TTL:    60 seconds
```

---

## 12. Bot Development Guide

### 12.1 Quick Start (Node.js)

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

const WEBHOOK_SECRET = process.env.WEBHOOK_SECRET;
const BOT_TOKEN = process.env.BOT_TOKEN;
const INCOMING_WEBHOOK_URL = process.env.INCOMING_WEBHOOK_URL;

// Verify webhook signature
function verifySignature(payload, signature, timestamp) {
  const signedPayload = `${timestamp}.${JSON.stringify(payload)}`;
  const expected = 'sha256=' + crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(signedPayload)
    .digest('hex');
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}

// Webhook handler
app.post('/webhook', (req, res) => {
  const signature = req.headers['x-webhook-signature'];
  const timestamp = req.headers['x-webhook-timestamp'];

  if (!verifySignature(req.body, signature, timestamp)) {
    return res.status(401).send('Invalid signature');
  }

  const event = req.body;

  switch (event.event_type) {
    case 'message.sent':
      handleMessage(event);
      break;
    case 'slash_command':
      handleSlashCommand(event, res);
      return;  // Response sent in handler
  }

  res.status(200).send('OK');
});

function handleMessage(event) {
  // React to messages mentioning the bot
  if (event.payload.mentions?.some(m => m.type === 'bot')) {
    sendMessage(event.channel.channel_id, 'Hello! How can I help?');
  }
}

function handleSlashCommand(event, res) {
  if (event.command === 'deploy') {
    // Immediate acknowledgment
    res.json({
      response_type: 'ephemeral',
      text: 'Starting deployment...'
    });

    // Async processing
    processDeployment(event).then(result => {
      // Send follow-up via response_url
      fetch(event.response_url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          response_type: 'in_channel',
          text: `Deployment complete: ${result}`
        })
      });
    });
  }
}

async function sendMessage(channelId, text) {
  await fetch(INCOMING_WEBHOOK_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ channel_id: channelId, text })
  });
}

app.listen(3000);
```

### 12.2 Quick Start (Python)

```python
import hmac
import hashlib
import os
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

WEBHOOK_SECRET = os.environ['WEBHOOK_SECRET']
BOT_TOKEN = os.environ['BOT_TOKEN']
INCOMING_WEBHOOK_URL = os.environ['INCOMING_WEBHOOK_URL']

def verify_signature(payload, signature, timestamp):
    signed_payload = f"{timestamp}.{payload}"
    expected = "sha256=" + hmac.new(
        WEBHOOK_SECRET.encode(),
        signed_payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Webhook-Signature')
    timestamp = request.headers.get('X-Webhook-Timestamp')

    if not verify_signature(request.data.decode(), signature, timestamp):
        return 'Invalid signature', 401

    event = request.json

    if event['event_type'] == 'message.sent':
        handle_message(event)
    elif event['event_type'] == 'slash_command':
        return handle_slash_command(event)

    return 'OK', 200

def handle_message(event):
    mentions = event.get('payload', {}).get('mentions', [])
    if any(m.get('type') == 'bot' for m in mentions):
        send_message(event['channel']['channel_id'], 'Hello! How can I help?')

def handle_slash_command(event):
    if event['command'] == 'status':
        return jsonify({
            'response_type': 'in_channel',
            'text': 'All systems operational ✅'
        })
    return 'OK', 200

def send_message(channel_id, text):
    requests.post(INCOMING_WEBHOOK_URL, json={
        'channel_id': channel_id,
        'text': text
    })

if __name__ == '__main__':
    app.run(port=3000)
```

### 12.3 Best Practices

1. **Always verify webhook signatures** — Never trust unverified payloads
2. **Respond quickly** — Return 200 within 3 seconds, process async
3. **Handle retries idempotently** — Use event_id to deduplicate
4. **Respect rate limits** — Implement exponential backoff
5. **Use ephemeral messages for errors** — Don't spam channels with error messages
6. **Store minimal state** — Prefer stateless designs where possible
7. **Log webhook events** — For debugging and audit trails

---

## Related Documents

- [Detailed Design: Webhook System](../detailed-design.md#3-webhook-system-design) — Webhook dispatcher architecture
- [Message Formats](./message-formats.md) — Adaptive cards and rich text
- [Authentication](./authentication.md) — User authentication (OIDC)
