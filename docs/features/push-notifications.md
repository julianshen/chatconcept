# Push Notifications

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Device Registration](#3-device-registration)
4. [Notification Triggers](#4-notification-triggers)
5. [Push Payload Formats](#5-push-payload-formats)
6. [User Preferences](#6-user-preferences)
7. [Badge Management](#7-badge-management)
8. [Rich Notifications](#8-rich-notifications)
9. [Push Worker Processing](#9-push-worker-processing)
10. [Delivery Tracking](#10-delivery-tracking)
11. [Rate Limiting & Batching](#11-rate-limiting--batching)

---

## 1. Overview

Push notifications deliver real-time alerts to mobile devices when users are offline or the app is in the background. The platform supports both **Apple Push Notification service (APNs)** for iOS and **Firebase Cloud Messaging (FCM)** for Android.

### Design Principles

1. **Relevance** — Only send notifications users care about
2. **Timeliness** — Deliver within seconds of the triggering event
3. **Actionable** — Include enough context to take action
4. **Respectful** — Honor user preferences and quiet hours
5. **Efficient** — Batch and deduplicate to reduce noise

### Notification Categories

| Category | Examples | Default |
|----------|----------|---------|
| **Direct Messages** | New DM, DM reply | Enabled |
| **Mentions** | @username, @here, @all | Enabled |
| **Thread Replies** | Reply to followed thread | Enabled |
| **Channel Activity** | New message in starred channel | Disabled |
| **Reactions** | Reaction to your message | Disabled |
| **System** | Security alerts, admin notices | Enabled |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Push Notification Architecture                          │
│                                                                              │
│  ┌──────────────┐                                                            │
│  │   MESSAGES   │                                                            │
│  │    Stream    │                                                            │
│  └──────┬───────┘                                                            │
│         │                                                                    │
│         │ message.sent, message.reaction, etc.                               │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Push Notification Evaluator                        │   │
│  │                                                                       │   │
│  │  For each event:                                                      │   │
│  │  1. Identify target users (mentions, DM recipients, thread followers) │   │
│  │  2. Filter by user preferences                                        │   │
│  │  3. Filter out online/active users (already see real-time)            │   │
│  │  4. Check quiet hours                                                 │   │
│  │  5. Deduplicate (collapse multiple events)                            │   │
│  │  6. Publish to NOTIFICATIONS stream                                   │   │
│  └──────────────────────────────────┬───────────────────────────────────┘   │
│                                     │                                        │
│                                     ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     NOTIFICATIONS Stream                              │   │
│  │                     (NATS JetStream)                                  │   │
│  └──────────────────────────────────┬───────────────────────────────────┘   │
│                                     │                                        │
│                                     │ Pull consumer                          │
│                                     ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Push Notification Worker Pool                     │   │
│  │                                                                       │   │
│  │  ┌─────────────────┐         ┌─────────────────┐                     │   │
│  │  │  APNs Sender    │         │   FCM Sender    │                     │   │
│  │  │                 │         │                 │                     │   │
│  │  │  • HTTP/2       │         │  • HTTP/2       │                     │   │
│  │  │  • JWT auth     │         │  • OAuth2       │                     │   │
│  │  │  • Connection   │         │  • Batch API    │                     │   │
│  │  │    pooling      │         │                 │                     │   │
│  │  └────────┬────────┘         └────────┬────────┘                     │   │
│  └───────────┼───────────────────────────┼──────────────────────────────┘   │
│              │                           │                                   │
│              ▼                           ▼                                   │
│       ┌─────────────┐             ┌─────────────┐                           │
│       │    APNs     │             │     FCM     │                           │
│       │   (Apple)   │             │  (Google)   │                           │
│       └──────┬──────┘             └──────┬──────┘                           │
│              │                           │                                   │
│              ▼                           ▼                                   │
│       ┌─────────────┐             ┌─────────────┐                           │
│       │  iOS Device │             │Android Device│                          │
│       └─────────────┘             └─────────────┘                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|---------------|
| **Push Evaluator** | Determine who should receive push, apply filters |
| **NOTIFICATIONS Stream** | Queue for pending push notifications |
| **Push Worker** | Deliver to APNs/FCM with retry logic |
| **Device Registry** | Store device tokens per user |
| **Preference Store** | User notification settings |

---

## 3. Device Registration

### 3.1 Register Device Token

Mobile apps register device tokens on startup and token refresh:

```
POST /api/v1/push/devices
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "platform": "ios",
  "token": "abc123def456...",
  "device_id": "device_01HZ3K...",
  "app_version": "3.2.1",
  "os_version": "17.2",
  "device_model": "iPhone 15 Pro",
  "locale": "en-US",
  "timezone": "America/Los_Angeles"
}
```

### 3.2 Response

```json
{
  "ok": true,
  "push_device_id": "push_01HZ3K...",
  "registered_at": "2026-02-01T10:30:00Z"
}
```

### 3.3 MongoDB: Device Token Schema

```javascript
// Collection: push_devices
{
  _id: ObjectId,
  push_device_id: "push_01HZ3K...",
  user_id: "usr_01HZ3K4M5N",

  // Platform
  platform: "ios",  // "ios" | "android"
  token: "abc123def456...",
  token_hash: "sha256:...",  // For deduplication

  // Device info
  device_id: "device_01HZ3K...",
  app_version: "3.2.1",
  os_version: "17.2",
  device_model: "iPhone 15 Pro",

  // Localization
  locale: "en-US",
  timezone: "America/Los_Angeles",

  // Status
  status: "active",  // "active" | "inactive" | "invalid"
  last_success_at: ISODate("2026-02-01T10:30:00Z"),
  last_failure_at: null,
  failure_count: 0,
  failure_reason: null,

  // Timestamps
  created_at: ISODate("2026-02-01T10:30:00Z"),
  updated_at: ISODate("2026-02-01T10:30:00Z")
}

// Indexes
{ push_device_id: 1 }           // unique
{ user_id: 1 }                  // find all devices for user
{ token_hash: 1 }               // unique, deduplication
{ platform: 1, status: 1 }      // platform-specific queries
{ status: 1, failure_count: 1 } // cleanup invalid tokens
```

### 3.4 Token Lifecycle

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Register  │────►│   Active    │────►│   Invalid   │
│   (new app) │     │             │     │  (expired)  │
└─────────────┘     └──────┬──────┘     └─────────────┘
                          │
                          │ Token refresh
                          ▼
                   ┌─────────────┐
                   │   Updated   │
                   │ (new token) │
                   └─────────────┘
```

### 3.5 Unregister Device

```
DELETE /api/v1/push/devices/{push_device_id}
Authorization: Bearer <access_token>
```

Called on logout or app uninstall detection.

---

## 4. Notification Triggers

### 4.1 Trigger Evaluation Flow

```go
func (e *PushEvaluator) EvaluateMessage(ctx context.Context, event *MessageEvent) error {
    // 1. Identify target users
    targets := e.identifyTargets(event)

    // 2. For each target, check if push should be sent
    for _, userID := range targets {
        shouldPush, reason := e.shouldSendPush(ctx, userID, event)
        if !shouldPush {
            e.metrics.PushSkipped(reason)
            continue
        }

        // 3. Create push notification
        notification := e.createNotification(userID, event)

        // 4. Publish to NOTIFICATIONS stream
        e.nats.Publish("notifications.push."+userID, notification)
    }

    return nil
}

func (e *PushEvaluator) identifyTargets(event *MessageEvent) []string {
    targets := make(map[string]struct{})

    // Direct mentions (@username)
    for _, mention := range event.Mentions {
        if mention.Type == "user" {
            targets[mention.UserID] = struct{}{}
        }
    }

    // @here - online users in channel
    if event.HasMention("here") {
        onlineUsers := e.getOnlineChannelMembers(event.ChannelID)
        for _, uid := range onlineUsers {
            targets[uid] = struct{}{}
        }
    }

    // @all - all channel members
    if event.HasMention("all") {
        allMembers := e.getChannelMembers(event.ChannelID)
        for _, uid := range allMembers {
            targets[uid] = struct{}{}
        }
    }

    // DM recipient
    if event.ChannelType == "dm" {
        for _, uid := range event.ChannelMembers {
            if uid != event.SenderID {
                targets[uid] = struct{}{}
            }
        }
    }

    // Thread followers (if thread reply)
    if event.ThreadID != "" {
        followers := e.getThreadFollowers(event.ThreadID)
        for _, uid := range followers {
            targets[uid] = struct{}{}
        }
    }

    // Remove sender
    delete(targets, event.SenderID)

    return mapKeys(targets)
}
```

### 4.2 Push Eligibility Checks

```go
func (e *PushEvaluator) shouldSendPush(ctx context.Context, userID string, event *MessageEvent) (bool, string) {
    // 1. Check if user has any registered devices
    devices := e.getActiveDevices(ctx, userID)
    if len(devices) == 0 {
        return false, "no_devices"
    }

    // 2. Check user preferences for this notification type
    prefs := e.getUserPreferences(ctx, userID)
    notifType := e.getNotificationType(event)
    if !prefs.IsEnabled(notifType) {
        return false, "preference_disabled"
    }

    // 3. Check channel-specific settings
    channelPrefs := prefs.GetChannelSettings(event.ChannelID)
    if channelPrefs.Muted {
        return false, "channel_muted"
    }

    // 4. Check if user is currently active (online and viewing)
    presence := e.getPresence(ctx, userID)
    if presence.Status == "online" && presence.ActiveChannel == event.ChannelID {
        return false, "user_active"
    }

    // 5. Check quiet hours
    if prefs.QuietHours.IsActive(time.Now(), prefs.Timezone) {
        return false, "quiet_hours"
    }

    // 6. Check deduplication window
    if e.recentlyNotified(ctx, userID, event.ChannelID) {
        return false, "deduplicated"
    }

    return true, ""
}
```

### 4.3 Notification Types and Triggers

| Type | Trigger | Default Priority |
|------|---------|-----------------|
| `dm_message` | New DM received | High |
| `dm_reply` | Reply in DM thread | High |
| `mention` | @username in message | High |
| `mention_here` | @here in channel | Normal |
| `mention_all` | @all in channel | Normal |
| `thread_reply` | Reply to followed thread | Normal |
| `channel_message` | New message in starred channel | Low |
| `reaction` | Reaction to your message | Low |
| `bot_dm` | Bot sends DM | Normal |
| `security_alert` | Login from new device, etc. | Critical |

---

## 5. Push Payload Formats

### 5.1 APNs Payload (iOS)

```json
{
  "aps": {
    "alert": {
      "title": "Alice Smith",
      "subtitle": "#general",
      "body": "Hey @bob, can you review the PR?"
    },
    "badge": 5,
    "sound": "default",
    "category": "MESSAGE",
    "thread-id": "ch_general",
    "mutable-content": 1,
    "interruption-level": "active"
  },
  "data": {
    "notification_id": "notif_01HZ3K...",
    "type": "mention",
    "channel_id": "ch_general",
    "channel_name": "general",
    "message_id": "msg_01HZ3K...",
    "thread_id": null,
    "sender_id": "usr_alice",
    "sender_name": "Alice Smith",
    "sender_avatar": "https://example.com/alice.jpg"
  }
}
```

### 5.2 FCM Payload (Android)

```json
{
  "message": {
    "token": "device_token_here",
    "notification": {
      "title": "Alice Smith",
      "body": "Hey @bob, can you review the PR?",
      "image": "https://example.com/alice.jpg"
    },
    "android": {
      "priority": "high",
      "notification": {
        "channel_id": "messages",
        "tag": "ch_general",
        "click_action": "OPEN_CHANNEL",
        "sound": "default",
        "notification_count": 5
      }
    },
    "data": {
      "notification_id": "notif_01HZ3K...",
      "type": "mention",
      "channel_id": "ch_general",
      "channel_name": "general",
      "message_id": "msg_01HZ3K...",
      "sender_id": "usr_alice",
      "sender_name": "Alice Smith"
    }
  }
}
```

### 5.3 Notification Content Templates

```go
type NotificationTemplates struct {
    templates map[string]NotificationTemplate
}

var templates = map[string]NotificationTemplate{
    "dm_message": {
        Title:    "{{.SenderName}}",
        Subtitle: "Direct Message",
        Body:     "{{.MessagePreview}}",
    },
    "mention": {
        Title:    "{{.SenderName}}",
        Subtitle: "#{{.ChannelName}}",
        Body:     "{{.MessagePreview}}",
    },
    "thread_reply": {
        Title:    "{{.SenderName}} replied",
        Subtitle: "Thread in #{{.ChannelName}}",
        Body:     "{{.MessagePreview}}",
    },
    "reaction": {
        Title:    "{{.SenderName}} reacted {{.Emoji}}",
        Subtitle: "#{{.ChannelName}}",
        Body:     "to: {{.OriginalMessagePreview}}",
    },
}

func (t *NotificationTemplates) Render(notifType string, data map[string]string) Notification {
    tmpl := t.templates[notifType]
    return Notification{
        Title:    renderTemplate(tmpl.Title, data),
        Subtitle: renderTemplate(tmpl.Subtitle, data),
        Body:     renderTemplate(tmpl.Body, data),
    }
}
```

### 5.4 Message Preview Truncation

```go
func truncateForPush(text string, maxLen int) string {
    // Strip markdown
    text = stripMarkdown(text)

    // Replace newlines with spaces
    text = strings.ReplaceAll(text, "\n", " ")

    // Truncate
    if len(text) > maxLen {
        return text[:maxLen-3] + "..."
    }
    return text
}

// iOS: 178 characters recommended for expanded view
// Android: 240 characters before truncation
const (
    iOSPreviewLength     = 178
    AndroidPreviewLength = 240
)
```

---

## 6. User Preferences

### 6.1 Notification Settings Schema

```javascript
// Embedded in users collection or separate collection
{
  user_id: "usr_01HZ3K4M5N",

  // Global settings
  push_enabled: true,
  email_enabled: true,

  // Notification types
  notifications: {
    dm_message: { push: true, email: false },
    dm_reply: { push: true, email: false },
    mention: { push: true, email: true },
    mention_here: { push: true, email: false },
    mention_all: { push: false, email: false },
    thread_reply: { push: true, email: false },
    channel_message: { push: false, email: false },
    reaction: { push: false, email: false },
    security_alert: { push: true, email: true }
  },

  // Quiet hours (Do Not Disturb)
  quiet_hours: {
    enabled: true,
    start: "22:00",
    end: "08:00",
    timezone: "America/Los_Angeles",
    allow_critical: true  // Security alerts bypass quiet hours
  },

  // Channel-specific overrides
  channel_settings: {
    "ch_general": { muted: false, notify_all: false },
    "ch_random": { muted: true, notify_all: false },
    "ch_alerts": { muted: false, notify_all: true }  // All messages
  },

  // Sound settings
  sound: {
    dm: "default",
    mention: "default",
    thread: "subtle"
  },

  updated_at: ISODate("2026-02-01T10:30:00Z")
}
```

### 6.2 Update Preferences API

```
PUT /api/v1/users/me/notifications
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "notifications": {
    "mention": { "push": true, "email": false }
  },
  "quiet_hours": {
    "enabled": true,
    "start": "23:00",
    "end": "07:00"
  }
}
```

### 6.3 Channel Mute/Unmute

```
POST /api/v1/channels/{channel_id}/mute
Authorization: Bearer <access_token>

{
  "muted": true,
  "duration": "1h"  // Optional: auto-unmute after duration
}
```

### 6.4 Redis: Preference Cache

```
Key:    push_prefs:{user_id}
Type:   Hash
Value:  {
  "push_enabled": "true",
  "dm_message": "true",
  "mention": "true",
  "quiet_start": "22:00",
  "quiet_end": "08:00",
  "timezone": "America/Los_Angeles"
}
TTL:    1 hour
```

---

## 7. Badge Management

### 7.1 Badge Count Calculation

Badge count represents total unread notifications:

```go
func (b *BadgeManager) CalculateBadge(ctx context.Context, userID string) int {
    // Sum of unread counts across:
    // - Unread DMs
    // - Unread mentions
    // - Unread thread replies (in followed threads)

    unreadDMs := b.redis.Get(ctx, "unread:dm:"+userID)
    unreadMentions := b.redis.Get(ctx, "unread:mentions:"+userID)
    unreadThreads := b.redis.Get(ctx, "unread:threads:"+userID)

    return unreadDMs + unreadMentions + unreadThreads
}
```

### 7.2 Badge Update Events

```go
// Increment badge on new notification
func (b *BadgeManager) IncrementBadge(ctx context.Context, userID string, notifType string) {
    key := "unread:" + notifType + ":" + userID
    b.redis.Incr(ctx, key)

    // Send silent push to update badge
    b.sendBadgeUpdate(ctx, userID)
}

// Clear badge when user views content
func (b *BadgeManager) ClearBadge(ctx context.Context, userID string, notifType string) {
    key := "unread:" + notifType + ":" + userID
    b.redis.Set(ctx, key, 0)

    // Send silent push to update badge
    b.sendBadgeUpdate(ctx, userID)
}
```

### 7.3 Silent Badge Update Push

```json
// APNs silent push for badge update
{
  "aps": {
    "badge": 3,
    "content-available": 1
  }
}
```

### 7.4 Redis: Badge Counters

```
Key:    unread:dm:{user_id}
Type:   String (integer)
Value:  count

Key:    unread:mentions:{user_id}
Type:   String (integer)
Value:  count

Key:    unread:threads:{user_id}
Type:   String (integer)
Value:  count

Key:    badge:{user_id}
Type:   String (integer)
Value:  total badge count (cached)
TTL:    none (cleared on read)
```

---

## 8. Rich Notifications

### 8.1 iOS Rich Notifications (Notification Service Extension)

Support for images, action buttons, and custom UI:

```json
{
  "aps": {
    "alert": {
      "title": "Alice Smith",
      "body": "Check out this screenshot"
    },
    "mutable-content": 1,
    "category": "MESSAGE_WITH_IMAGE"
  },
  "data": {
    "image_url": "https://cdn.example.com/img/abc123.jpg",
    "message_id": "msg_01HZ3K..."
  }
}
```

### 8.2 Action Buttons

```json
// iOS categories defined in app
{
  "categories": [
    {
      "identifier": "MESSAGE",
      "actions": [
        {
          "identifier": "REPLY",
          "title": "Reply",
          "options": ["foreground"]
        },
        {
          "identifier": "MARK_READ",
          "title": "Mark as Read",
          "options": ["destructive"]
        }
      ]
    },
    {
      "identifier": "DM",
      "actions": [
        {
          "identifier": "REPLY",
          "title": "Reply",
          "textInput": {
            "buttonTitle": "Send",
            "placeholder": "Type a reply..."
          }
        }
      ]
    }
  ]
}
```

### 8.3 Inline Reply Handling

```
POST /api/v1/push/actions
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "notification_id": "notif_01HZ3K...",
  "action": "REPLY",
  "text_input": "Thanks, I'll check it out!",
  "device_id": "device_01HZ3K..."
}
```

### 8.4 Android Notification Channels

```kotlin
// Android app setup
val channels = listOf(
    NotificationChannel(
        "dm_messages",
        "Direct Messages",
        NotificationManager.IMPORTANCE_HIGH
    ),
    NotificationChannel(
        "mentions",
        "Mentions",
        NotificationManager.IMPORTANCE_HIGH
    ),
    NotificationChannel(
        "threads",
        "Thread Replies",
        NotificationManager.IMPORTANCE_DEFAULT
    ),
    NotificationChannel(
        "channels",
        "Channel Messages",
        NotificationManager.IMPORTANCE_LOW
    )
)
```

---

## 9. Push Worker Processing

### 9.1 NATS Consumer Configuration

```
Stream: NOTIFICATIONS
  Subjects: notifications.push.>
  Retention: WorkQueue
  Max Age: 7 days
  Replicas: 3

Consumer: push-worker-pool
  Type: Pull, Durable
  Ack Policy: Explicit
  Max Ack Pending: 1000
  Max Deliver: 5
  Backoff: 1s, 5s, 30s, 2m, 10m
```

### 9.2 Push Worker Implementation

```go
func (w *PushWorker) Process(ctx context.Context, msg *nats.Msg) error {
    var notification PushNotification
    json.Unmarshal(msg.Data, &notification)

    // 1. Get user's devices
    devices, err := w.getActiveDevices(ctx, notification.UserID)
    if err != nil {
        return err
    }

    // 2. Send to each device
    var lastErr error
    for _, device := range devices {
        var err error
        switch device.Platform {
        case "ios":
            err = w.sendAPNs(ctx, device, notification)
        case "android":
            err = w.sendFCM(ctx, device, notification)
        }

        if err != nil {
            lastErr = err
            w.handleDeliveryFailure(ctx, device, err)
        } else {
            w.recordDeliverySuccess(ctx, device, notification)
        }
    }

    return lastErr
}
```

### 9.3 APNs Sender

```go
type APNsSender struct {
    client *http.Client
    keyID  string
    teamID string
    key    *ecdsa.PrivateKey
}

func (s *APNsSender) Send(ctx context.Context, device *Device, notif *PushNotification) error {
    // 1. Build payload
    payload := s.buildPayload(notif)

    // 2. Create request
    url := fmt.Sprintf("https://api.push.apple.com/3/device/%s", device.Token)
    req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(payload))

    // 3. Set headers
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("apns-topic", "com.example.chat")
    req.Header.Set("apns-push-type", "alert")
    req.Header.Set("apns-priority", "10")
    req.Header.Set("apns-expiration", "0")
    req.Header.Set("authorization", "bearer "+s.generateJWT())

    // 4. Send request
    resp, err := s.client.Do(req)
    if err != nil {
        return fmt.Errorf("APNs request failed: %w", err)
    }
    defer resp.Body.Close()

    // 5. Handle response
    if resp.StatusCode != http.StatusOK {
        var apnsErr APNsError
        json.NewDecoder(resp.Body).Decode(&apnsErr)
        return s.handleAPNsError(device, apnsErr)
    }

    return nil
}

func (s *APNsSender) handleAPNsError(device *Device, err APNsError) error {
    switch err.Reason {
    case "BadDeviceToken", "Unregistered":
        // Mark device as invalid
        s.invalidateDevice(device)
        return nil  // Don't retry
    case "TooManyRequests":
        return fmt.Errorf("rate limited")  // Will retry
    default:
        return fmt.Errorf("APNs error: %s", err.Reason)
    }
}
```

### 9.4 FCM Sender

```go
type FCMSender struct {
    client    *http.Client
    projectID string
    creds     *google.Credentials
}

func (s *FCMSender) Send(ctx context.Context, device *Device, notif *PushNotification) error {
    // 1. Build FCM message
    message := s.buildMessage(device, notif)

    // 2. Create request
    url := fmt.Sprintf("https://fcm.googleapis.com/v1/projects/%s/messages:send", s.projectID)
    body, _ := json.Marshal(map[string]interface{}{"message": message})
    req, _ := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))

    // 3. Set OAuth2 token
    token, _ := s.creds.TokenSource.Token()
    req.Header.Set("Authorization", "Bearer "+token.AccessToken)
    req.Header.Set("Content-Type", "application/json")

    // 4. Send request
    resp, err := s.client.Do(req)
    if err != nil {
        return fmt.Errorf("FCM request failed: %w", err)
    }
    defer resp.Body.Close()

    // 5. Handle response
    if resp.StatusCode != http.StatusOK {
        var fcmErr FCMError
        json.NewDecoder(resp.Body).Decode(&fcmErr)
        return s.handleFCMError(device, fcmErr)
    }

    return nil
}

func (s *FCMSender) handleFCMError(device *Device, err FCMError) error {
    switch err.Error.Status {
    case "UNREGISTERED":
        s.invalidateDevice(device)
        return nil
    case "QUOTA_EXCEEDED":
        return fmt.Errorf("rate limited")
    default:
        return fmt.Errorf("FCM error: %s", err.Error.Message)
    }
}
```

---

## 10. Delivery Tracking

### 10.1 Delivery Status

| Status | Description |
|--------|-------------|
| `pending` | Queued in NOTIFICATIONS stream |
| `sent` | Delivered to APNs/FCM |
| `delivered` | Confirmed delivered to device |
| `failed` | Permanent failure |
| `expired` | TTL exceeded before delivery |

### 10.2 Delivery Record Schema

```javascript
// Collection: push_delivery_log
{
  _id: ObjectId,
  notification_id: "notif_01HZ3K...",
  user_id: "usr_01HZ3K4M5N",
  device_id: "push_01HZ3K...",

  // Notification details
  type: "mention",
  channel_id: "ch_general",
  message_id: "msg_01HZ3K...",

  // Delivery status
  status: "sent",
  platform: "ios",
  apns_id: "abc-123-def",  // APNs response ID

  // Timing
  created_at: ISODate("2026-02-01T10:30:00.000Z"),
  sent_at: ISODate("2026-02-01T10:30:00.123Z"),
  delivered_at: null,

  // Errors
  error_code: null,
  error_message: null,
  retry_count: 0
}

// Indexes
{ notification_id: 1 }
{ user_id: 1, created_at: -1 }
{ status: 1, created_at: 1 }  // For cleanup
{ created_at: 1 }             // TTL index (30 days)
```

### 10.3 Delivery Confirmation (Optional)

Apps can report delivery confirmation:

```
POST /api/v1/push/delivered
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "notification_id": "notif_01HZ3K...",
  "delivered_at": "2026-02-01T10:30:01.456Z"
}
```

---

## 11. Rate Limiting & Batching

### 11.1 Rate Limits

| Platform | Limit | Period |
|----------|-------|--------|
| APNs | No hard limit | Per connection |
| FCM | 1,000,000 | Per minute |

### 11.2 Notification Collapsing

Collapse multiple notifications for the same context:

```go
func (e *PushEvaluator) shouldCollapse(ctx context.Context, userID, channelID string) bool {
    key := fmt.Sprintf("push_recent:%s:%s", userID, channelID)

    // Check if notified in last 30 seconds
    exists, _ := e.redis.Exists(ctx, key)
    if exists {
        return true
    }

    // Mark as recently notified
    e.redis.Set(ctx, key, "1", 30*time.Second)
    return false
}
```

### 11.3 Batching for High Volume

For channels with many members, batch notifications:

```go
func (w *PushWorker) ProcessBatch(ctx context.Context, notifications []*PushNotification) error {
    // Group by platform and send in batches
    iosNotifs := filterByPlatform(notifications, "ios")
    androidNotifs := filterByPlatform(notifications, "android")

    // FCM supports batch sends (up to 500)
    for _, batch := range chunk(androidNotifs, 500) {
        w.fcm.SendBatch(ctx, batch)
    }

    // APNs uses HTTP/2 multiplexing
    for _, notif := range iosNotifs {
        w.apns.Send(ctx, notif)
    }

    return nil
}
```

### 11.4 Redis: Deduplication

```
Key:    push_recent:{user_id}:{channel_id}
Type:   String
Value:  "1"
TTL:    30 seconds

Key:    push_sent:{notification_id}
Type:   String
Value:  "1"
TTL:    5 minutes (idempotency)
```

---

## Related Documents

- [Detailed Design: Push Worker](../detailed-design.md#24-push-notification-worker) — Worker architecture
- [Authentication](./authentication.md) — User authentication for device registration
- [Multi-Device Support](./authentication.md#11-multi-device-support) — Device management
