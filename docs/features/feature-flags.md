# Feature Flags System

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Flag Types](#3-flag-types)
4. [Flag Configuration](#4-flag-configuration)
5. [Targeting Rules](#5-targeting-rules)
6. [SDK Integration](#6-sdk-integration)
7. [Operational Flags](#7-operational-flags)
8. [Rollout Strategies](#8-rollout-strategies)
9. [Admin Interface](#9-admin-interface)
10. [Audit & Compliance](#10-audit--compliance)
11. [Best Practices](#11-best-practices)

---

## 1. Overview

The feature flags system enables controlled rollouts, A/B testing, and operational kill switches across all platform services. It provides a centralized mechanism to toggle features without deployment.

### Use Cases

| Use Case | Example |
|----------|---------|
| **Gradual Rollout** | Roll out new message editor to 5% → 25% → 100% |
| **Beta Testing** | Enable threads V2 for beta users only |
| **Kill Switch** | Disable file uploads during incident |
| **Ops Control** | Toggle regional autonomy mode |
| **A/B Testing** | Test two search ranking algorithms |
| **Entitlements** | Enable premium features for paid users |

### Design Principles

1. **Fast Evaluation** — Flag checks < 1ms, cached locally
2. **Consistent** — Same user sees same variant across sessions
3. **Observable** — All flag evaluations logged for debugging
4. **Fail-Safe** — Default to safe behavior when flag service unavailable
5. **Auditable** — All changes tracked with who/when/why

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Feature Flags Architecture                            │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         Flag Configuration                            │   │
│  │                           (MongoDB)                                   │   │
│  │                                                                       │   │
│  │  • Flag definitions                                                   │   │
│  │  • Targeting rules                                                    │   │
│  │  • Rollout percentages                                                │   │
│  │  • Audit history                                                      │   │
│  └────────────────────────────────┬─────────────────────────────────────┘   │
│                                   │                                          │
│                                   │ Sync (every 30s)                         │
│                                   ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Flag Distribution Service                        │   │
│  │                                                                       │   │
│  │  • Serves flag configurations                                         │   │
│  │  • Pushes updates via SSE/WebSocket                                   │   │
│  │  • Caches in Redis for fast reads                                     │   │
│  └────────────────────────────────┬─────────────────────────────────────┘   │
│                                   │                                          │
│            ┌──────────────────────┼──────────────────────┐                  │
│            │                      │                      │                  │
│            ▼                      ▼                      ▼                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │  Command Service │  │  Query Service   │  │  Notification    │          │
│  │                  │  │                  │  │    Service       │          │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │          │
│  │  │ Flag SDK   │  │  │  │ Flag SDK   │  │  │  │ Flag SDK   │  │          │
│  │  │            │  │  │  │            │  │  │  │            │  │          │
│  │  │ • Cache    │  │  │  │ • Cache    │  │  │  │ • Cache    │  │          │
│  │  │ • Evaluate │  │  │  │ • Evaluate │  │  │  │ • Evaluate │  │          │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         Client Applications                           │   │
│  │                                                                       │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │   │
│  │  │  Web Client │    │ iOS Client  │    │Android Client│               │   │
│  │  │             │    │             │    │              │               │   │
│  │  │ Flag SDK    │    │ Flag SDK    │    │ Flag SDK     │               │   │
│  │  │ (JS)        │    │ (Swift)     │    │ (Kotlin)     │               │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Responsibility |
|-----------|---------------|
| **Flag Configuration Store** | MongoDB collection storing flag definitions |
| **Flag Distribution Service** | Serves configs, pushes updates, caches in Redis |
| **Server SDK** | Go library for server-side evaluation |
| **Client SDK** | JS/Swift/Kotlin libraries for client-side evaluation |
| **Admin UI** | Web interface for flag management |

---

## 3. Flag Types

### 3.1 Boolean Flags

Simple on/off toggle.

```json
{
  "key": "enable_message_reactions",
  "type": "boolean",
  "default_value": false,
  "description": "Enable emoji reactions on messages"
}
```

### 3.2 String Flags

Return a string value (useful for A/B variants).

```json
{
  "key": "search_algorithm",
  "type": "string",
  "default_value": "bm25",
  "variants": ["bm25", "semantic", "hybrid"],
  "description": "Search ranking algorithm"
}
```

### 3.3 Number Flags

Return numeric values (thresholds, limits).

```json
{
  "key": "max_file_upload_mb",
  "type": "number",
  "default_value": 100,
  "description": "Maximum file upload size in MB"
}
```

### 3.4 JSON Flags

Return complex configuration objects.

```json
{
  "key": "rate_limit_config",
  "type": "json",
  "default_value": {
    "messages_per_minute": 60,
    "api_calls_per_minute": 100,
    "burst_allowance": 10
  },
  "description": "Rate limiting configuration"
}
```

---

## 4. Flag Configuration

### 4.1 MongoDB Schema

```javascript
// Collection: feature_flags
{
  _id: ObjectId,
  key: "enable_threads_v2",

  // Flag metadata
  name: "Threads V2",
  description: "New threaded conversation UI",
  type: "boolean",

  // Values
  default_value: false,

  // For string/number flags with variants
  variants: [
    { value: true, name: "Enabled" },
    { value: false, name: "Disabled" }
  ],

  // Targeting rules (evaluated in order)
  rules: [
    {
      id: "rule_1",
      name: "Beta Users",
      conditions: [
        { attribute: "user.tags", operator: "contains", value: "beta" }
      ],
      serve: { value: true },
      enabled: true
    },
    {
      id: "rule_2",
      name: "Gradual Rollout",
      conditions: [],  // No conditions = percentage rollout
      serve: { percentage: { true: 25, false: 75 } },
      enabled: true
    }
  ],

  // Fallback when no rules match
  fallthrough: {
    serve: { value: false }
  },

  // Off variation (when flag is disabled)
  off_variation: false,

  // Flag status
  enabled: true,
  archived: false,

  // Ownership
  owner: "usr_alice",
  team: "platform",
  tags: ["frontend", "ux", "beta"],

  // Lifecycle
  created_at: ISODate("2026-02-01T10:00:00Z"),
  updated_at: ISODate("2026-02-01T15:30:00Z"),
  created_by: "usr_alice",
  updated_by: "usr_bob"
}

// Indexes
{ key: 1 }          // unique
{ tags: 1 }
{ team: 1 }
{ enabled: 1 }
{ "rules.id": 1 }
```

### 4.2 Redis Cache Structure

```
# Flag configuration cache
Key:    flags:config
Type:   Hash
Field:  {flag_key}
Value:  {JSON serialized flag config}
TTL:    none (updated via pub/sub)

# Flag evaluation cache (per-user)
Key:    flags:eval:{user_id}
Type:   Hash
Field:  {flag_key}
Value:  {evaluated_value}
TTL:    5 minutes

# Flag version for cache invalidation
Key:    flags:version
Type:   String
Value:  {version_number}
```

### 4.3 Configuration Sync

```go
// Flag Distribution Service - sync loop
func (s *FlagService) StartSync(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.syncFlags(ctx)
        }
    }
}

func (s *FlagService) syncFlags(ctx context.Context) {
    // Get current version
    currentVersion, _ := s.redis.Get(ctx, "flags:version").Int64()

    // Check MongoDB for updates
    cursor, _ := s.mongo.Find(ctx, "feature_flags", bson.M{
        "updated_at": bson.M{"$gt": s.lastSync},
    })

    var updatedFlags []Flag
    cursor.All(ctx, &updatedFlags)

    if len(updatedFlags) > 0 {
        // Update Redis cache
        pipe := s.redis.Pipeline()
        for _, flag := range updatedFlags {
            data, _ := json.Marshal(flag)
            pipe.HSet(ctx, "flags:config", flag.Key, data)
        }
        pipe.Incr(ctx, "flags:version")
        pipe.Exec(ctx)

        // Notify connected SDKs
        s.notifyUpdate(ctx, updatedFlags)

        s.lastSync = time.Now()
    }
}
```

---

## 5. Targeting Rules

### 5.1 Condition Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `equals` | Exact match | `user.plan equals "premium"` |
| `not_equals` | Not equal | `user.status not_equals "suspended"` |
| `contains` | Array contains | `user.tags contains "beta"` |
| `not_contains` | Array doesn't contain | `user.tags not_contains "excluded"` |
| `starts_with` | String prefix | `user.email starts_with "test"` |
| `ends_with` | String suffix | `user.email ends_with "@example.com"` |
| `matches` | Regex match | `user.id matches "^usr_test"` |
| `in` | Value in list | `user.country in ["US", "CA", "UK"]` |
| `not_in` | Value not in list | `user.country not_in ["CN", "RU"]` |
| `greater_than` | Numeric comparison | `user.account_age_days greater_than 30` |
| `less_than` | Numeric comparison | `user.message_count less_than 100` |
| `semver_equals` | Version comparison | `app.version semver_equals "2.0.0"` |
| `semver_greater` | Version comparison | `app.version semver_greater "1.9.0"` |

### 5.2 User Context Attributes

```go
// Standard user context for evaluation
type EvaluationContext struct {
    // User attributes
    User struct {
        ID          string            `json:"id"`
        Email       string            `json:"email"`
        Username    string            `json:"username"`
        Plan        string            `json:"plan"`        // "free", "pro", "enterprise"
        Roles       []string          `json:"roles"`
        Tags        []string          `json:"tags"`        // ["beta", "internal"]
        CreatedAt   time.Time         `json:"created_at"`
        Country     string            `json:"country"`
        Custom      map[string]any    `json:"custom"`
    } `json:"user"`

    // Device/client attributes
    Device struct {
        Type        string            `json:"type"`        // "web", "ios", "android"
        OS          string            `json:"os"`
        OSVersion   string            `json:"os_version"`
        AppVersion  string            `json:"app_version"`
        Locale      string            `json:"locale"`
    } `json:"device"`

    // Request attributes
    Request struct {
        IP          string            `json:"ip"`
        Country     string            `json:"country"`
        Region      string            `json:"region"`
    } `json:"request"`

    // Custom attributes
    Custom map[string]any `json:"custom"`
}
```

### 5.3 Example Rules

**Beta Users Only:**
```json
{
  "conditions": [
    { "attribute": "user.tags", "operator": "contains", "value": "beta" }
  ],
  "serve": { "value": true }
}
```

**Enterprise Plan + US Region:**
```json
{
  "conditions": [
    { "attribute": "user.plan", "operator": "equals", "value": "enterprise" },
    { "attribute": "request.country", "operator": "equals", "value": "US" }
  ],
  "serve": { "value": true }
}
```

**Gradual Rollout by User ID:**
```json
{
  "conditions": [],
  "serve": {
    "percentage": {
      "true": 25,
      "false": 75
    },
    "bucket_by": "user.id"
  }
}
```

**A/B Test with 3 Variants:**
```json
{
  "conditions": [],
  "serve": {
    "percentage": {
      "control": 34,
      "variant_a": 33,
      "variant_b": 33
    },
    "bucket_by": "user.id"
  }
}
```

### 5.4 Percentage Bucketing

Consistent hashing ensures users always see the same variant:

```go
func (e *Evaluator) getBucket(flagKey, bucketBy string, ctx *EvaluationContext) int {
    // Get the bucketing value
    bucketValue := e.getAttribute(ctx, bucketBy)
    if bucketValue == "" {
        bucketValue = ctx.User.ID
    }

    // Hash flag key + bucket value for consistent assignment
    hashInput := flagKey + ":" + bucketValue
    hash := sha256.Sum256([]byte(hashInput))

    // Convert first 4 bytes to int and mod 100
    bucket := int(binary.BigEndian.Uint32(hash[:4])) % 100

    return bucket
}

func (e *Evaluator) evaluatePercentage(flag *Flag, rule *Rule, ctx *EvaluationContext) any {
    bucket := e.getBucket(flag.Key, rule.Serve.BucketBy, ctx)

    cumulative := 0
    for variant, percentage := range rule.Serve.Percentage {
        cumulative += percentage
        if bucket < cumulative {
            return variant
        }
    }

    return flag.DefaultValue
}
```

---

## 6. SDK Integration

### 6.1 Server SDK (Go)

```go
package flags

import (
    "context"
    "encoding/json"
    "sync"
    "time"
)

type Client struct {
    config     *Config
    cache      map[string]*Flag
    cacheMu    sync.RWMutex
    redis      *redis.Client
    evaluator  *Evaluator
    updateChan chan FlagUpdate
}

// Initialize client
func NewClient(config *Config) (*Client, error) {
    client := &Client{
        config:     config,
        cache:      make(map[string]*Flag),
        evaluator:  NewEvaluator(),
        updateChan: make(chan FlagUpdate, 100),
    }

    // Initial sync
    if err := client.sync(context.Background()); err != nil {
        return nil, err
    }

    // Start background sync
    go client.startUpdateListener()

    return client, nil
}

// Evaluate boolean flag
func (c *Client) BoolVariation(ctx context.Context, key string, evalCtx *EvaluationContext, defaultVal bool) bool {
    value, _ := c.evaluate(ctx, key, evalCtx, defaultVal)
    if b, ok := value.(bool); ok {
        return b
    }
    return defaultVal
}

// Evaluate string flag
func (c *Client) StringVariation(ctx context.Context, key string, evalCtx *EvaluationContext, defaultVal string) string {
    value, _ := c.evaluate(ctx, key, evalCtx, defaultVal)
    if s, ok := value.(string); ok {
        return s
    }
    return defaultVal
}

// Evaluate with details (for debugging/analytics)
func (c *Client) BoolVariationDetail(ctx context.Context, key string, evalCtx *EvaluationContext, defaultVal bool) (*EvaluationDetail, error) {
    return c.evaluateWithDetail(ctx, key, evalCtx, defaultVal)
}

// Core evaluation logic
func (c *Client) evaluate(ctx context.Context, key string, evalCtx *EvaluationContext, defaultVal any) (any, error) {
    // Get flag from cache
    c.cacheMu.RLock()
    flag, exists := c.cache[key]
    c.cacheMu.RUnlock()

    if !exists {
        c.logEvaluation(key, evalCtx, defaultVal, "FLAG_NOT_FOUND")
        return defaultVal, ErrFlagNotFound
    }

    // Check if flag is enabled
    if !flag.Enabled {
        c.logEvaluation(key, evalCtx, flag.OffVariation, "FLAG_DISABLED")
        return flag.OffVariation, nil
    }

    // Evaluate rules
    value, reason := c.evaluator.Evaluate(flag, evalCtx)
    c.logEvaluation(key, evalCtx, value, reason)

    return value, nil
}
```

**Usage in Services:**

```go
// In Command Service
func (h *Handler) SendMessage(ctx context.Context, req *SendMessageRequest) error {
    evalCtx := flags.ContextFromRequest(ctx)

    // Check if new message format is enabled
    if h.flags.BoolVariation(ctx, "enable_rich_text_v2", evalCtx, false) {
        // Use new rich text parser
        req.Text = h.richTextV2.Parse(req.Text)
    } else {
        // Use legacy parser
        req.Text = h.richText.Parse(req.Text)
    }

    // Check rate limit from flag
    rateLimit := h.flags.IntVariation(ctx, "message_rate_limit", evalCtx, 60)
    if !h.rateLimiter.Allow(req.UserID, rateLimit) {
        return ErrRateLimited
    }

    // ...
}
```

### 6.2 Client SDK (JavaScript)

```typescript
// flags-client.ts
export class FlagsClient {
  private config: FlagsConfig;
  private cache: Map<string, FlagValue> = new Map();
  private context: EvaluationContext;
  private eventSource: EventSource | null = null;

  constructor(config: FlagsConfig) {
    this.config = config;
    this.context = config.context;
  }

  async initialize(): Promise<void> {
    // Fetch initial flags
    const response = await fetch(`${this.config.baseUrl}/flags`, {
      headers: {
        'Authorization': `Bearer ${this.config.clientKey}`,
        'X-Context': JSON.stringify(this.context),
      },
    });

    const flags = await response.json();
    this.updateCache(flags);

    // Subscribe to updates
    this.connectSSE();
  }

  private connectSSE(): void {
    this.eventSource = new EventSource(
      `${this.config.baseUrl}/flags/stream?context=${encodeURIComponent(JSON.stringify(this.context))}`
    );

    this.eventSource.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.cache.set(update.key, update.value);
      this.config.onUpdate?.(update.key, update.value);
    };
  }

  boolVariation(key: string, defaultValue: boolean): boolean {
    const value = this.cache.get(key);
    return typeof value === 'boolean' ? value : defaultValue;
  }

  stringVariation(key: string, defaultValue: string): string {
    const value = this.cache.get(key);
    return typeof value === 'string' ? value : defaultValue;
  }

  // Update context (e.g., after login)
  async updateContext(context: Partial<EvaluationContext>): Promise<void> {
    this.context = { ...this.context, ...context };
    await this.initialize(); // Re-fetch with new context
  }
}

// Usage in React
const flags = new FlagsClient({
  baseUrl: 'https://api.example.com',
  clientKey: 'client_key_xxx',
  context: {
    user: { id: 'anonymous' },
    device: { type: 'web' },
  },
});

function MessageComposer() {
  const [richTextEnabled, setRichTextEnabled] = useState(
    flags.boolVariation('enable_rich_text_v2', false)
  );

  useEffect(() => {
    flags.config.onUpdate = (key, value) => {
      if (key === 'enable_rich_text_v2') {
        setRichTextEnabled(value as boolean);
      }
    };
  }, []);

  return richTextEnabled ? <RichTextEditorV2 /> : <RichTextEditor />;
}
```

### 6.3 Bootstrapping Flags

For fast initial render, include flag values in HTML:

```html
<!-- Server renders initial flags -->
<script>
  window.__FLAGS__ = {
    "enable_rich_text_v2": true,
    "search_algorithm": "hybrid",
    "max_file_upload_mb": 100
  };
</script>
```

```typescript
// Client uses bootstrap, then syncs
const flags = new FlagsClient({
  bootstrap: window.__FLAGS__,
  // ...
});
```

---

## 7. Operational Flags

### 7.1 Kill Switches

Predefined flags for emergency operations:

| Flag | Description | Default |
|------|-------------|---------|
| `kill_file_uploads` | Disable all file uploads | `false` |
| `kill_push_notifications` | Disable push notifications | `false` |
| `kill_webhooks_outgoing` | Disable outgoing webhooks | `false` |
| `kill_search` | Disable search functionality | `false` |
| `kill_message_send` | Disable sending messages (read-only mode) | `false` |

### 7.2 Ops Control Flags

| Flag | Description | Default |
|------|-------------|---------|
| `enable_regional_autonomy` | Enable regional autonomous mode | `false` |
| `enable_maintenance_mode` | Show maintenance banner | `false` |
| `rate_limit_multiplier` | Adjust rate limits (0.5 = stricter, 2.0 = relaxed) | `1.0` |
| `cassandra_read_consistency` | Override read consistency | `"LOCAL_ONE"` |
| `nats_ack_wait_seconds` | NATS consumer ack wait time | `30` |

### 7.3 Using Kill Switches

```go
func (h *Handler) UploadFile(ctx context.Context, req *UploadRequest) error {
    evalCtx := flags.SystemContext() // No user context for kill switch

    // Check kill switch
    if h.flags.BoolVariation(ctx, "kill_file_uploads", evalCtx, false) {
        return ErrServiceUnavailable("File uploads temporarily disabled")
    }

    // Check user-specific flag for upload size
    userCtx := flags.ContextFromRequest(ctx)
    maxSize := h.flags.IntVariation(ctx, "max_file_upload_mb", userCtx, 100)

    if req.Size > maxSize*1024*1024 {
        return ErrFileTooLarge
    }

    // ...
}
```

---

## 8. Rollout Strategies

### 8.1 Canary Rollout

```yaml
# Phase 1: Internal testing (1%)
rules:
  - name: "Internal Users"
    conditions:
      - attribute: "user.email"
        operator: "ends_with"
        value: "@example.com"
    serve:
      value: true

# Phase 2: Beta users (5%)
rules:
  - name: "Beta Users"
    conditions:
      - attribute: "user.tags"
        operator: "contains"
        value: "beta"
    serve:
      value: true

# Phase 3: Gradual rollout
rules:
  - name: "Gradual Rollout"
    conditions: []
    serve:
      percentage:
        true: 5
        false: 95
```

### 8.2 Ring-Based Rollout

```yaml
# Ring 0: Developers (immediate)
# Ring 1: Internal employees (day 1)
# Ring 2: Beta users (day 3)
# Ring 3: 10% users (day 5)
# Ring 4: 50% users (day 7)
# Ring 5: 100% users (day 10)

rules:
  - name: "Ring 0 - Developers"
    conditions:
      - attribute: "user.roles"
        operator: "contains"
        value: "developer"
    serve:
      value: true

  - name: "Ring 1 - Internal"
    conditions:
      - attribute: "user.email"
        operator: "ends_with"
        value: "@example.com"
    serve:
      value: true

  - name: "Ring 2 - Beta"
    conditions:
      - attribute: "user.tags"
        operator: "contains"
        value: "beta"
    serve:
      value: true

  - name: "Ring 3-5 - Percentage"
    conditions: []
    serve:
      percentage:
        true: 10   # Increase to 50, then 100
        false: 90
```

### 8.3 Scheduled Rollout

```go
// Flag Distribution Service - scheduled rollout
func (s *FlagService) processScheduledRollouts(ctx context.Context) {
    now := time.Now()

    cursor, _ := s.mongo.Find(ctx, "flag_schedules", bson.M{
        "scheduled_at": bson.M{"$lte": now},
        "executed": false,
    })

    var schedules []FlagSchedule
    cursor.All(ctx, &schedules)

    for _, schedule := range schedules {
        // Apply scheduled change
        s.mongo.UpdateOne(ctx, "feature_flags",
            bson.M{"key": schedule.FlagKey},
            bson.M{"$set": schedule.Changes},
        )

        // Mark as executed
        s.mongo.UpdateOne(ctx, "flag_schedules",
            bson.M{"_id": schedule.ID},
            bson.M{"$set": bson.M{"executed": true, "executed_at": now}},
        )

        // Audit log
        s.auditLog(ctx, "SCHEDULED_ROLLOUT", schedule)
    }
}
```

---

## 9. Admin Interface

### 9.1 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/admin/flags` | GET | List all flags |
| `/api/v1/admin/flags` | POST | Create flag |
| `/api/v1/admin/flags/{key}` | GET | Get flag details |
| `/api/v1/admin/flags/{key}` | PUT | Update flag |
| `/api/v1/admin/flags/{key}` | DELETE | Archive flag |
| `/api/v1/admin/flags/{key}/toggle` | POST | Enable/disable flag |
| `/api/v1/admin/flags/{key}/rules` | PUT | Update targeting rules |
| `/api/v1/admin/flags/{key}/audit` | GET | Get change history |
| `/api/v1/admin/flags/{key}/evaluate` | POST | Test evaluation |

### 9.2 Create Flag

```
POST /api/v1/admin/flags
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "key": "enable_threads_v2",
  "name": "Threads V2",
  "description": "New threaded conversation UI",
  "type": "boolean",
  "default_value": false,
  "tags": ["frontend", "ux"],
  "team": "platform"
}
```

### 9.3 Update Rollout Percentage

```
PUT /api/v1/admin/flags/enable_threads_v2/rules
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "rules": [
    {
      "id": "gradual_rollout",
      "name": "Gradual Rollout",
      "conditions": [],
      "serve": {
        "percentage": {
          "true": 50,
          "false": 50
        }
      }
    }
  ]
}
```

### 9.4 Test Evaluation

```
POST /api/v1/admin/flags/enable_threads_v2/evaluate
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "context": {
    "user": {
      "id": "usr_test123",
      "email": "test@example.com",
      "plan": "pro",
      "tags": ["beta"]
    }
  }
}

Response:
{
  "value": true,
  "reason": "RULE_MATCH",
  "rule_id": "beta_users",
  "rule_name": "Beta Users"
}
```

---

## 10. Audit & Compliance

### 10.1 Audit Log Schema

```javascript
// Collection: flag_audit_log
{
  _id: ObjectId,
  flag_key: "enable_threads_v2",
  action: "UPDATE_RULES",  // CREATE, UPDATE, DELETE, TOGGLE, UPDATE_RULES

  actor: {
    user_id: "usr_alice",
    username: "alice",
    ip_address: "192.168.1.100"
  },

  changes: {
    before: { /* previous state */ },
    after: { /* new state */ }
  },

  reason: "Increasing rollout to 50% after successful 25% test",

  timestamp: ISODate("2026-02-01T15:30:00Z")
}

// Indexes
{ flag_key: 1, timestamp: -1 }
{ "actor.user_id": 1, timestamp: -1 }
{ timestamp: -1 }  // TTL index optional
```

### 10.2 Change Approval Workflow

For production flags, require approval:

```go
type FlagChangeRequest struct {
    ID          string
    FlagKey     string
    RequestedBy string
    Changes     FlagChanges
    Reason      string
    Status      string // "pending", "approved", "rejected"
    ApprovedBy  string
    CreatedAt   time.Time
}

func (s *FlagService) RequestChange(ctx context.Context, req *FlagChangeRequest) error {
    flag, _ := s.getFlag(ctx, req.FlagKey)

    // Check if flag requires approval
    if flag.RequiresApproval {
        req.Status = "pending"
        s.mongo.Insert(ctx, "flag_change_requests", req)

        // Notify approvers
        s.notifyApprovers(ctx, req)
        return nil
    }

    // Apply immediately
    return s.applyChange(ctx, req)
}
```

### 10.3 Evaluation Logging

```go
func (c *Client) logEvaluation(key string, ctx *EvaluationContext, value any, reason string) {
    // Sample 1% of evaluations for analytics
    if rand.Float64() > 0.01 {
        return
    }

    log := EvaluationLog{
        FlagKey:   key,
        UserID:    ctx.User.ID,
        Value:     value,
        Reason:    reason,
        Timestamp: time.Now(),
    }

    // Async publish to analytics
    c.analyticsQueue <- log
}
```

---

## 11. Best Practices

### 11.1 Naming Conventions

```
# Boolean flags: enable_*, disable_*, show_*, hide_*
enable_threads_v2
enable_rich_text_editor
show_typing_indicators
disable_file_previews

# Kill switches: kill_*
kill_file_uploads
kill_push_notifications

# Ops flags: ops_*
ops_maintenance_mode
ops_read_only_mode

# Experiment flags: exp_*
exp_search_algorithm
exp_onboarding_flow

# Limit flags: limit_*, max_*, min_*
max_file_upload_mb
limit_api_rate
```

### 11.2 Flag Lifecycle

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Created │────►│ Testing │────►│ Rollout │────►│ Stable  │────►│Archived │
└─────────┘     └─────────┘     └─────────┘     └─────────┘     └─────────┘
                                                     │
                                                     │ 100% for 30 days
                                                     ▼
                                               Remove from code
```

### 11.3 Technical Debt Prevention

1. **Set expiration dates** — Flags should have removal deadlines
2. **Track flag age** — Alert on flags > 90 days old
3. **Code cleanup** — Remove flag checks when 100% rolled out
4. **Documentation** — Link flags to design docs/tickets

### 11.4 Performance Guidelines

| Guideline | Recommendation |
|-----------|----------------|
| Cache refresh | 30 seconds server-side, 60 seconds client-side |
| Evaluation time | < 1ms per flag |
| Max rules per flag | 20 |
| Max conditions per rule | 10 |
| Bootstrap flags | Include critical flags in HTML |

---

## Related Documents

- [Operations Guide](../operations/operations-guide.md) — Kill switch procedures
- [API Reference](../api/api-reference.md) — Admin API endpoints
- [Implementation Plan](../appendices/implementation-plan.md) — Rollout phases
