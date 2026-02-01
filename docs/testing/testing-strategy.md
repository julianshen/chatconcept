# Testing Strategy

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Testing Philosophy](#1-testing-philosophy)
2. [Test Pyramid](#2-test-pyramid)
3. [Test Types](#3-test-types)
4. [Component Testing Matrix](#4-component-testing-matrix)
5. [Test Environments](#5-test-environments)
6. [Test Data Management](#6-test-data-management)
7. [CI/CD Integration](#7-cicd-integration)
8. [Quality Gates](#8-quality-gates)
9. [Contract Testing](#9-contract-testing)
10. [Chaos Engineering](#10-chaos-engineering)
11. [Production Testing](#11-production-testing)
12. [Test Tooling](#12-test-tooling)

---

## 1. Testing Philosophy

### 1.1 Core Principles

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Testing Philosophy                                   │
│                                                                              │
│  1. SHIFT LEFT                                                               │
│     Catch bugs early. Unit tests run in milliseconds, production bugs       │
│     cost hours. Every bug caught in CI saves 10x the cost of production.    │
│                                                                              │
│  2. TEST BEHAVIOR, NOT IMPLEMENTATION                                        │
│     Tests should verify what the system does, not how it does it.           │
│     Refactoring should not break tests.                                     │
│                                                                              │
│  3. FAST FEEDBACK                                                            │
│     PR builds complete in < 10 minutes. Slow tests kill velocity.           │
│     Parallelize everything. Mock external dependencies.                      │
│                                                                              │
│  4. PRODUCTION PARITY                                                        │
│     Staging mirrors production. Same infrastructure, same scale.            │
│     If it works in staging, it works in production.                         │
│                                                                              │
│  5. OBSERVABLE TESTING                                                       │
│     Every test failure produces actionable diagnostics.                     │
│     Logs, traces, and metrics are captured for failed tests.                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Testing Goals

| Goal | Target | Current |
|------|--------|---------|
| Unit test coverage | > 80% | — |
| Integration test coverage | > 60% | — |
| E2E critical path coverage | 100% | — |
| PR build time | < 10 min | — |
| Main branch build time | < 30 min | — |
| Flaky test rate | < 1% | — |
| Mean time to detect (MTTD) | < 5 min | — |

---

## 2. Test Pyramid

```
                            ┌─────────────────┐
                            │   Manual/       │  ← Exploratory testing
                            │   Exploratory   │    Usability testing
                            └────────┬────────┘    Production verification
                                     │
                         ┌───────────┴───────────┐
                         │     E2E Tests         │  ← Critical user journeys
                         │     (~50 tests)       │    Cross-service flows
                         │     ~15 min           │    Real browsers (Playwright)
                         └───────────┬───────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    │      Integration Tests          │  ← Service boundaries
                    │      (~500 tests)               │    Database interactions
                    │      ~5 min                     │    NATS message flows
                    └────────────────┬────────────────┘
                                     │
           ┌─────────────────────────┴─────────────────────────┐
           │                   Unit Tests                       │  ← Business logic
           │                   (~5000 tests)                    │    Pure functions
           │                   ~2 min                           │    Isolated components
           └───────────────────────────────────────────────────┘

                    ▲                                        ▲
                    │                                        │
               More                                     More
            Integration                              Isolation
               Slower                                 Faster
```

### 2.1 Test Distribution

| Layer | Count | Execution Time | Frequency |
|-------|-------|----------------|-----------|
| Unit | ~5,000 | < 2 min | Every commit |
| Integration | ~500 | < 5 min | Every PR |
| E2E | ~50 | < 15 min | Every merge to main |
| Performance | ~20 | 2-24 hours | Nightly / Weekly |
| Chaos | ~10 | 1-4 hours | Weekly |

---

## 3. Test Types

### 3.1 Unit Tests

**Purpose:** Verify individual functions and methods in isolation.

```go
// Example: Command Service message validation
func TestValidateMessagePayload(t *testing.T) {
    tests := []struct {
        name    string
        payload MessagePayload
        wantErr error
    }{
        {
            name: "valid plain text message",
            payload: MessagePayload{
                ChannelID: "ch_general",
                Text:      "Hello, world!",
            },
            wantErr: nil,
        },
        {
            name: "empty text rejected",
            payload: MessagePayload{
                ChannelID: "ch_general",
                Text:      "",
            },
            wantErr: ErrEmptyMessage,
        },
        {
            name: "text exceeds max length",
            payload: MessagePayload{
                ChannelID: "ch_general",
                Text:      strings.Repeat("a", 50001),
            },
            wantErr: ErrMessageTooLong,
        },
        {
            name: "missing channel ID",
            payload: MessagePayload{
                Text: "Hello",
            },
            wantErr: ErrMissingChannelID,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateMessagePayload(tt.payload)
            if !errors.Is(err, tt.wantErr) {
                t.Errorf("ValidateMessagePayload() error = %v, want %v", err, tt.wantErr)
            }
        })
    }
}
```

**Guidelines:**
- Test one thing per test
- Use table-driven tests
- Mock external dependencies
- No network calls, no database
- Run in < 10ms per test

### 3.2 Integration Tests

**Purpose:** Verify service interactions with real dependencies.

```go
// Example: Message Writer integration with Cassandra
func TestMessageWriter_PersistMessage(t *testing.T) {
    // Setup: Use testcontainers for Cassandra
    ctx := context.Background()
    cassandra := testcontainers.NewCassandraContainer(ctx)
    defer cassandra.Terminate(ctx)

    session := cassandra.Session()
    writer := NewMessageWriter(session)

    // Test
    msg := &Message{
        ID:        "msg_test_123",
        ChannelID: "ch_general",
        SenderID:  "usr_alice",
        Text:      "Integration test message",
        CreatedAt: time.Now(),
    }

    err := writer.Persist(ctx, msg)
    require.NoError(t, err)

    // Verify
    var retrieved Message
    err = session.Query(`
        SELECT message_id, channel_id, sender_id, text
        FROM messages
        WHERE channel_id = ? AND bucket = ? AND message_id = ?
    `, msg.ChannelID, msg.Bucket(), msg.ID).Scan(
        &retrieved.ID, &retrieved.ChannelID, &retrieved.SenderID, &retrieved.Text,
    )
    require.NoError(t, err)
    assert.Equal(t, msg.Text, retrieved.Text)
}
```

**Guidelines:**
- Use testcontainers for databases (Cassandra, MongoDB, Redis)
- Use embedded NATS for message bus tests
- Test happy path + error scenarios
- Verify side effects (database writes, events published)
- Clean up test data

### 3.3 End-to-End Tests

**Purpose:** Verify complete user journeys across all services.

```typescript
// Example: Playwright E2E test for message sending
import { test, expect } from '@playwright/test';

test.describe('Message Flow', () => {
  test('user can send and receive messages', async ({ browser }) => {
    // Setup: Two users in same channel
    const aliceContext = await browser.newContext();
    const bobContext = await browser.newContext();

    const alicePage = await aliceContext.newPage();
    const bobPage = await bobContext.newPage();

    // Alice logs in and opens channel
    await alicePage.goto('/login');
    await alicePage.fill('[data-testid="email"]', 'alice@test.com');
    await alicePage.fill('[data-testid="password"]', 'testpass123');
    await alicePage.click('[data-testid="login-button"]');
    await alicePage.waitForURL('/channels');
    await alicePage.click('[data-testid="channel-general"]');

    // Bob logs in and opens same channel
    await bobPage.goto('/login');
    await bobPage.fill('[data-testid="email"]', 'bob@test.com');
    await bobPage.fill('[data-testid="password"]', 'testpass123');
    await bobPage.click('[data-testid="login-button"]');
    await bobPage.waitForURL('/channels');
    await bobPage.click('[data-testid="channel-general"]');

    // Alice sends message
    const testMessage = `E2E test message ${Date.now()}`;
    await alicePage.fill('[data-testid="message-input"]', testMessage);
    await alicePage.press('[data-testid="message-input"]', 'Enter');

    // Verify Alice sees her message
    await expect(alicePage.locator(`text=${testMessage}`)).toBeVisible({ timeout: 5000 });

    // Verify Bob receives message in real-time
    await expect(bobPage.locator(`text=${testMessage}`)).toBeVisible({ timeout: 5000 });

    // Verify message persisted (page refresh)
    await bobPage.reload();
    await bobPage.click('[data-testid="channel-general"]');
    await expect(bobPage.locator(`text=${testMessage}`)).toBeVisible({ timeout: 5000 });
  });

  test('typing indicator appears for other users', async ({ browser }) => {
    // ... similar setup ...

    // Alice starts typing
    await alicePage.fill('[data-testid="message-input"]', 'Hello');

    // Bob sees typing indicator
    await expect(bobPage.locator('[data-testid="typing-indicator"]')).toContainText('Alice is typing');

    // Alice stops typing (clear input)
    await alicePage.fill('[data-testid="message-input"]', '');

    // Bob typing indicator disappears
    await expect(bobPage.locator('[data-testid="typing-indicator"]')).not.toBeVisible({ timeout: 6000 });
  });
});
```

**Guidelines:**
- Test critical user journeys only
- Use realistic test data
- Test across browsers (Chrome, Firefox, Safari)
- Include mobile viewport tests
- Capture screenshots on failure

### 3.4 Performance Tests

**Purpose:** Verify system meets performance SLOs under load.

See [Fan-Out Performance Testing](./fanout-performance-testing.md) for detailed strategy.

**Test Categories:**

| Type | Purpose | Duration | Frequency |
|------|---------|----------|-----------|
| Baseline | Establish performance baseline | 30 min | Weekly |
| Load | Verify production-level load | 2 hours | Nightly |
| Stress | Find breaking points | 1 hour | Weekly |
| Spike | Test burst traffic handling | 30 min | Weekly |
| Soak | Find memory leaks | 24 hours | Weekly |

### 3.5 Security Tests

**Purpose:** Identify security vulnerabilities before production.

```yaml
# Security test pipeline
security_tests:
  # Static analysis
  - name: SAST (Semgrep)
    tool: semgrep
    config: p/golang, p/typescript, p/security-audit

  # Dependency scanning
  - name: Dependency Check
    tool: trivy
    targets:
      - go.mod
      - package.json
      - Dockerfile

  # Container scanning
  - name: Container Scan
    tool: trivy
    targets:
      - ghcr.io/org/command-service:latest
      - ghcr.io/org/query-service:latest

  # Dynamic testing
  - name: DAST (ZAP)
    tool: zap
    target: https://staging.chat.example.com
    scan_type: full

  # API security
  - name: API Security
    tool: nuclei
    templates:
      - cves/
      - exposures/
      - misconfiguration/
```

**Security Test Checklist:**

- [ ] Authentication bypass attempts
- [ ] Authorization (BOLA, BFLA)
- [ ] Injection (SQL, NoSQL, Command)
- [ ] XSS (Stored, Reflected, DOM)
- [ ] CSRF protection
- [ ] Rate limiting effectiveness
- [ ] JWT validation
- [ ] WebSocket security
- [ ] File upload validation
- [ ] Sensitive data exposure

### 3.6 Accessibility Tests

**Purpose:** Ensure platform is accessible to users with disabilities.

```typescript
// Playwright accessibility tests with axe-core
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility', () => {
  test('channel page has no accessibility violations', async ({ page }) => {
    await page.goto('/channels/general');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    expect(results.violations).toEqual([]);
  });

  test('message input is keyboard accessible', async ({ page }) => {
    await page.goto('/channels/general');

    // Tab to message input
    await page.keyboard.press('Tab');
    await page.keyboard.press('Tab');
    await page.keyboard.press('Tab');

    // Verify focus is on message input
    const focused = await page.locator(':focus');
    await expect(focused).toHaveAttribute('data-testid', 'message-input');

    // Type and send with keyboard
    await page.keyboard.type('Keyboard accessible message');
    await page.keyboard.press('Enter');

    await expect(page.locator('text=Keyboard accessible message')).toBeVisible();
  });
});
```

---

## 4. Component Testing Matrix

### 4.1 Service-Level Testing

| Service | Unit | Integration | E2E | Performance | Security |
|---------|------|-------------|-----|-------------|----------|
| **API Gateway** | Config validation | Auth token validation | Full request flow | Rate limiting, latency | Auth bypass, injection |
| **Command Service** | Validation, preprocessing | NATS publishing | Send message | Throughput | Input validation |
| **Query Service** | Query building | DB queries | Fetch history | Query latency | Data access control |
| **Fan-Out Service** | Routing logic | NATS consume/publish | Message delivery | Fan-out throughput | — |
| **Notification Service** | Connection handling | WebSocket + Redis | Real-time delivery | Connection scaling | WebSocket security |
| **Auth Service** | Token validation | Keycloak integration | Login/logout flow | Auth latency | Auth security |
| **Message Writer** | Serialization | Cassandra writes | — | Write throughput | — |
| **Search Indexer** | Text processing | Elasticsearch writes | Search results | Index latency | Query injection |
| **Push Worker** | Payload building | APNs/FCM integration | Push delivery | Push throughput | Token security |
| **Webhook Dispatcher** | Signature generation | HTTP delivery | Webhook delivery | Retry handling | SSRF prevention |

### 4.2 Cross-Service Testing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Cross-Service Test Scenarios                             │
│                                                                              │
│  1. MESSAGE SEND → DELIVERY                                                  │
│     Client → Gateway → Command → NATS → Fan-Out → Notif → Client            │
│     Verify: < 100ms E2E latency                                              │
│                                                                              │
│  2. MESSAGE SEND → PERSISTENCE → QUERY                                       │
│     Client → Command → NATS → Writer → Cassandra                            │
│     Client → Gateway → Query → Cassandra → Client                           │
│     Verify: Message readable within 500ms of send                           │
│                                                                              │
│  3. USER LOGIN → PRESENCE → NOTIFICATION                                     │
│     Client → Auth → Keycloak → JWT                                          │
│     Client → Notif (WebSocket) → Redis (presence)                           │
│     Verify: User appears online to others within 2s                         │
│                                                                              │
│  4. WEBHOOK REGISTRATION → DELIVERY                                          │
│     Admin → Command → MongoDB (webhook config)                              │
│     Message → Fan-Out → Webhook Worker → External Bot                       │
│     Verify: Webhook delivered with valid signature                          │
│                                                                              │
│  5. MULTI-DEVICE SYNC                                                        │
│     Desktop reads message → Redis (read pointer)                            │
│     NATS (user.sync) → Mobile badge update                                  │
│     Verify: Badge syncs within 2s                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Test Environments

### 5.1 Environment Tiers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Environment Tiers                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  LOCAL (Developer Machine)                                           │    │
│  │  • Docker Compose with all services                                 │    │
│  │  • Seeded test data                                                 │    │
│  │  • Hot reload enabled                                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  CI (Ephemeral, per-PR)                                              │    │
│  │  • Kubernetes namespace per PR                                      │    │
│  │  • Testcontainers for databases                                     │    │
│  │  • Isolated NATS cluster                                            │    │
│  │  • Auto-destroyed after tests                                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  STAGING (Persistent)                                                │    │
│  │  • Production-like infrastructure                                   │    │
│  │  • Scaled-down replicas (1/10th of prod)                            │    │
│  │  • Synthetic test data (no real user data)                          │    │
│  │  • E2E and performance tests run here                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  PRODUCTION                                                          │    │
│  │  • Canary deployments (5% → 25% → 100%)                             │    │
│  │  • Synthetic monitoring (heartbeat tests)                           │    │
│  │  • Feature flags for gradual rollout                                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Local Development Environment

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  # Core infrastructure
  nats:
    image: nats:2.10-alpine
    command: ["--jetstream", "--store_dir=/data"]
    ports:
      - "4222:4222"
      - "8222:8222"

  cassandra:
    image: cassandra:4.1
    ports:
      - "9042:9042"
    environment:
      - CASSANDRA_CLUSTER_NAME=test
    healthcheck:
      test: ["CMD", "cqlsh", "-e", "describe keyspaces"]
      interval: 10s
      timeout: 5s
      retries: 10

  mongodb:
    image: mongo:7.0
    ports:
      - "27017:27017"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  elasticsearch:
    image: elasticsearch:8.11.0
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false

  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    ports:
      - "8080:8080"
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    command: start-dev

  # Test data seeder
  seeder:
    build: ./tests/seeder
    depends_on:
      cassandra:
        condition: service_healthy
      mongodb:
        condition: service_started
    environment:
      - CASSANDRA_HOST=cassandra
      - MONGODB_URI=mongodb://mongodb:27017
```

### 5.3 CI Environment (GitHub Actions)

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  pull_request:
  push:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Run unit tests
        run: go test -race -coverprofile=coverage.out ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.out

  integration-tests:
    runs-on: ubuntu-latest
    services:
      nats:
        image: nats:2.10-alpine
        options: --health-cmd "wget -q --spider http://localhost:8222/healthz" --health-interval 5s
        ports:
          - 4222:4222

    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Run integration tests
        run: go test -tags=integration -race ./...
        env:
          NATS_URL: nats://localhost:4222
          TESTCONTAINERS_RYUK_DISABLED: true

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Start services
        run: docker compose -f docker-compose.test.yml up -d

      - name: Wait for services
        run: ./scripts/wait-for-services.sh

      - name: Run E2E tests
        run: npx playwright test

      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

---

## 6. Test Data Management

### 6.1 Test Data Categories

| Category | Description | Storage | Lifecycle |
|----------|-------------|---------|-----------|
| **Fixtures** | Static test data (users, channels) | Git repository | Versioned |
| **Generated** | Dynamic data (messages, events) | Generated at runtime | Per-test |
| **Snapshots** | Expected outputs | Git repository | Versioned |
| **Synthetic** | Production-like load data | S3 bucket | Updated monthly |

### 6.2 Test Data Factory

```go
// tests/factory/factory.go
package factory

import (
    "fmt"
    "time"

    "github.com/brianvoe/gofakeit/v6"
)

type Factory struct {
    faker *gofakeit.Faker
    seq   int64
}

func New() *Factory {
    return &Factory{
        faker: gofakeit.New(0),
    }
}

func (f *Factory) User() *User {
    f.seq++
    return &User{
        ID:          fmt.Sprintf("usr_%08d", f.seq),
        Email:       f.faker.Email(),
        DisplayName: f.faker.Name(),
        Avatar:      f.faker.ImageURL(100, 100),
        CreatedAt:   time.Now().Add(-time.Duration(f.faker.IntRange(1, 365)) * 24 * time.Hour),
    }
}

func (f *Factory) Channel(opts ...ChannelOption) *Channel {
    f.seq++
    ch := &Channel{
        ID:          fmt.Sprintf("ch_%08d", f.seq),
        Name:        f.faker.BuzzWord(),
        Description: f.faker.Sentence(10),
        CreatedAt:   time.Now(),
        MemberCount: f.faker.IntRange(2, 100),
    }
    for _, opt := range opts {
        opt(ch)
    }
    return ch
}

func (f *Factory) Message(channelID, senderID string) *Message {
    f.seq++
    return &Message{
        ID:        fmt.Sprintf("msg_%08d", f.seq),
        ChannelID: channelID,
        SenderID:  senderID,
        Text:      f.faker.Sentence(f.faker.IntRange(3, 20)),
        CreatedAt: time.Now(),
    }
}

// Options pattern for customization
type ChannelOption func(*Channel)

func WithMemberCount(count int) ChannelOption {
    return func(c *Channel) {
        c.MemberCount = count
    }
}

func WithName(name string) ChannelOption {
    return func(c *Channel) {
        c.Name = name
    }
}
```

### 6.3 Seed Data Script

```go
// tests/seeder/main.go
package main

func main() {
    // Connect to databases
    cassandra := connectCassandra()
    mongodb := connectMongoDB()
    redis := connectRedis()

    // Seed users
    users := seedUsers(mongodb, 1000)
    log.Printf("Seeded %d users", len(users))

    // Seed channels with memberships
    channels := seedChannels(mongodb, users, 100)
    log.Printf("Seeded %d channels", len(channels))

    // Seed messages
    messages := seedMessages(cassandra, channels, users, 10000)
    log.Printf("Seeded %d messages", len(messages))

    // Seed read pointers
    seedReadPointers(redis, users, channels)
    log.Printf("Seeded read pointers")

    log.Println("Seed complete!")
}

func seedUsers(db *mongo.Database, count int) []*User {
    factory := factory.New()
    users := make([]*User, count)

    for i := 0; i < count; i++ {
        users[i] = factory.User()
    }

    // Bulk insert
    docs := make([]interface{}, len(users))
    for i, u := range users {
        docs[i] = u
    }
    db.Collection("users").InsertMany(context.Background(), docs)

    return users
}
```

---

## 7. CI/CD Integration

### 7.1 Pipeline Stages

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CI/CD Pipeline                                     │
│                                                                              │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐       │
│  │  Lint   │──►│  Unit   │──►│ Integ   │──►│  Build  │──►│  E2E    │       │
│  │         │   │  Tests  │   │  Tests  │   │  Image  │   │  Tests  │       │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘       │
│     ~1min        ~2min         ~5min         ~3min         ~15min           │
│                                                                              │
│                                    │                                         │
│                                    ▼                                         │
│                         ┌─────────────────────┐                             │
│                         │   Security Scan     │                             │
│                         │   (parallel)        │                             │
│                         └─────────────────────┘                             │
│                                    │                                         │
│                    ┌───────────────┼───────────────┐                        │
│                    ▼               ▼               ▼                        │
│             ┌───────────┐   ┌───────────┐   ┌───────────┐                   │
│             │  Deploy   │   │  Perf     │   │  Deploy   │                   │
│             │  Staging  │   │  Tests    │   │  Canary   │                   │
│             └───────────┘   │ (nightly) │   │  (5%)     │                   │
│                             └───────────┘   └─────┬─────┘                   │
│                                                   │                          │
│                                    ┌──────────────┴──────────────┐          │
│                                    ▼                             ▼          │
│                             ┌───────────┐                 ┌───────────┐     │
│                             │  Monitor  │────────────────►│  Full     │     │
│                             │  Canary   │  if healthy     │  Rollout  │     │
│                             └───────────┘                 └───────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Test Parallelization

```yaml
# Parallel test execution strategy
test_parallelization:
  unit_tests:
    strategy: by-package
    max_parallel: 8
    timeout: 5m

  integration_tests:
    strategy: by-service
    services:
      - command-service
      - query-service
      - fanout-service
      - notification-service
    max_parallel: 4
    timeout: 10m

  e2e_tests:
    strategy: by-file
    shards: 4
    timeout: 20m
```

### 7.3 Test Result Reporting

```yaml
# Test reporting configuration
reporting:
  # JUnit XML for CI systems
  junit:
    enabled: true
    output: test-results/junit.xml

  # Coverage reporting
  coverage:
    format: [html, lcov, cobertura]
    output: coverage/
    thresholds:
      statements: 80
      branches: 70
      functions: 80
      lines: 80

  # Allure for detailed reports
  allure:
    enabled: true
    output: allure-results/

  # Slack notifications
  notifications:
    slack:
      channel: '#ci-alerts'
      on_failure: true
      on_success: false
```

---

## 8. Quality Gates

### 8.1 PR Quality Gates

```yaml
# Required checks for PR merge
pr_quality_gates:
  required:
    - lint-pass
    - unit-tests-pass
    - integration-tests-pass
    - coverage-threshold-met
    - security-scan-pass
    - no-new-vulnerabilities

  coverage:
    min_total: 80%
    min_diff: 70%  # New code must have 70%+ coverage

  performance:
    # Only for PRs touching performance-critical paths
    paths:
      - 'services/fanout/**'
      - 'services/notification/**'
    checks:
      - baseline-comparison
      - no-regression
```

### 8.2 Release Quality Gates

```yaml
# Required for production release
release_quality_gates:
  required:
    - all-pr-gates-pass
    - e2e-tests-pass
    - staging-smoke-tests-pass
    - performance-baseline-met
    - security-audit-pass
    - accessibility-scan-pass

  performance:
    comparison: last-release
    max_latency_increase: 10%
    max_throughput_decrease: 5%

  security:
    max_critical: 0
    max_high: 0
    max_medium: 5
```

### 8.3 Gate Enforcement

```go
// cmd/gate-check/main.go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type GateResult struct {
    Gate    string `json:"gate"`
    Passed  bool   `json:"passed"`
    Message string `json:"message"`
}

func main() {
    results := []GateResult{
        checkCoverage(),
        checkSecurityScan(),
        checkPerformance(),
        checkLint(),
    }

    allPassed := true
    for _, r := range results {
        if !r.Passed {
            allPassed = false
            fmt.Printf("❌ %s: %s\n", r.Gate, r.Message)
        } else {
            fmt.Printf("✅ %s: %s\n", r.Gate, r.Message)
        }
    }

    // Output for CI
    output, _ := json.Marshal(results)
    os.WriteFile("gate-results.json", output, 0644)

    if !allPassed {
        os.Exit(1)
    }
}

func checkCoverage() GateResult {
    coverage := getCoverageFromReport()
    threshold := 80.0

    return GateResult{
        Gate:    "coverage",
        Passed:  coverage >= threshold,
        Message: fmt.Sprintf("%.1f%% (threshold: %.1f%%)", coverage, threshold),
    }
}
```

---

## 9. Contract Testing

### 9.1 API Contract Testing (OpenAPI)

```yaml
# openapi/chat-api.yaml
openapi: 3.0.3
info:
  title: Chat Platform API
  version: 1.0.0

paths:
  /api/v1/chat.sendMessage:
    post:
      operationId: sendMessage
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SendMessageRequest'
      responses:
        '202':
          description: Message accepted
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SendMessageResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  schemas:
    SendMessageRequest:
      type: object
      required:
        - channel_id
        - text
      properties:
        channel_id:
          type: string
          pattern: '^ch_[a-z0-9]+$'
        text:
          type: string
          maxLength: 50000
        thread_id:
          type: string
          pattern: '^msg_[a-z0-9]+$'
```

```typescript
// Contract test using Prism
import { createHttpClient } from '@stoplight/prism-http';

describe('API Contract Tests', () => {
  const prism = createHttpClient({
    spec: './openapi/chat-api.yaml',
    validateRequest: true,
    validateResponse: true,
  });

  test('sendMessage accepts valid request', async () => {
    const response = await prism.request({
      method: 'POST',
      url: '/api/v1/chat.sendMessage',
      headers: { 'Content-Type': 'application/json' },
      body: {
        channel_id: 'ch_general',
        text: 'Hello, world!',
      },
    });

    expect(response.status).toBe(202);
  });

  test('sendMessage rejects invalid channel_id', async () => {
    const response = await prism.request({
      method: 'POST',
      url: '/api/v1/chat.sendMessage',
      body: {
        channel_id: 'invalid',  // Missing ch_ prefix
        text: 'Hello',
      },
    });

    expect(response.status).toBe(400);
  });
});
```

### 9.2 Event Contract Testing (AsyncAPI)

```yaml
# asyncapi/messages.yaml
asyncapi: 2.6.0
info:
  title: Chat Platform Events
  version: 1.0.0

channels:
  messages.send.{channelId}:
    parameters:
      channelId:
        schema:
          type: string
    publish:
      message:
        $ref: '#/components/messages/MessageSent'

components:
  messages:
    MessageSent:
      payload:
        type: object
        required:
          - event_id
          - type
          - channel_id
          - sender_id
          - content
          - timestamp
        properties:
          event_id:
            type: string
            format: uuid
          type:
            type: string
            const: message.sent
          channel_id:
            type: string
          sender_id:
            type: string
          content:
            $ref: '#/components/schemas/MessageContent'
          timestamp:
            type: string
            format: date-time
```

```go
// Event contract test
func TestMessageSentEventContract(t *testing.T) {
    // Load AsyncAPI spec
    spec := loadAsyncAPISpec("asyncapi/messages.yaml")
    schema := spec.Components.Messages["MessageSent"].Payload

    // Test valid event
    validEvent := map[string]interface{}{
        "event_id":   "evt_123",
        "type":       "message.sent",
        "channel_id": "ch_general",
        "sender_id":  "usr_alice",
        "content": map[string]interface{}{
            "text": "Hello!",
        },
        "timestamp": time.Now().Format(time.RFC3339),
    }

    err := validateAgainstSchema(validEvent, schema)
    assert.NoError(t, err)

    // Test invalid event (missing required field)
    invalidEvent := map[string]interface{}{
        "event_id": "evt_123",
        "type":     "message.sent",
        // Missing channel_id
    }

    err = validateAgainstSchema(invalidEvent, schema)
    assert.Error(t, err)
}
```

---

## 10. Chaos Engineering

### 10.1 Chaos Experiments

```yaml
# chaos/experiments.yaml
experiments:
  - name: nats-node-failure
    description: Kill one NATS node and verify cluster recovers
    hypothesis: "Messages continue to be delivered with < 5s interruption"
    method:
      - action: pod-kill
        target:
          selector: app=nats
          count: 1
        duration: 5m
    steadyState:
      - probe: message-delivery-latency
        tolerance: p99 < 100ms

  - name: cassandra-latency-injection
    description: Add 500ms latency to Cassandra reads
    hypothesis: "Query service returns cached results or degrades gracefully"
    method:
      - action: network-latency
        target:
          selector: app=cassandra
        latency: 500ms
        duration: 10m
    steadyState:
      - probe: query-service-latency
        tolerance: p99 < 2s

  - name: redis-failover
    description: Trigger Redis Sentinel failover
    hypothesis: "Presence updates continue within 30s"
    method:
      - action: pod-kill
        target:
          selector: app=redis,role=master
        count: 1
    steadyState:
      - probe: presence-update-success
        tolerance: rate > 95%

  - name: fan-out-cpu-pressure
    description: Limit Fan-Out CPU to 50%
    hypothesis: "Consumer lag increases but recovers after pressure removed"
    method:
      - action: cpu-pressure
        target:
          selector: app=fanout
        load: 50
        duration: 15m
    steadyState:
      - probe: jetstream-consumer-lag
        tolerance: lag < 50000
```

### 10.2 Chaos Toolkit Implementation

```python
# chaos/experiment_nats_failure.py
from chaoslib.experiment import run_experiment

experiment = {
    "title": "NATS Node Failure Recovery",
    "description": "Verify message delivery continues when a NATS node fails",

    "steady-state-hypothesis": {
        "title": "Messages are delivered within SLO",
        "probes": [
            {
                "name": "message-delivery-p99",
                "type": "probe",
                "provider": {
                    "type": "python",
                    "module": "probes.prometheus",
                    "func": "query_latency_percentile",
                    "arguments": {
                        "metric": "message_delivery_latency_seconds",
                        "percentile": 0.99
                    }
                },
                "tolerance": {
                    "type": "range",
                    "range": [0, 0.1]  # < 100ms
                }
            }
        ]
    },

    "method": [
        {
            "name": "kill-nats-node",
            "type": "action",
            "provider": {
                "type": "python",
                "module": "chaosk8s.pod.actions",
                "func": "terminate_pods",
                "arguments": {
                    "label_selector": "app=nats",
                    "qty": 1,
                    "rand": True
                }
            },
            "pauses": {
                "after": 60  # Wait 60s for recovery
            }
        }
    ],

    "rollbacks": [
        {
            "name": "ensure-nats-healthy",
            "type": "action",
            "provider": {
                "type": "python",
                "module": "probes.nats",
                "func": "wait_for_cluster_healthy",
                "arguments": {
                    "timeout": 300
                }
            }
        }
    ]
}

if __name__ == "__main__":
    run_experiment(experiment)
```

---

## 11. Production Testing

### 11.1 Synthetic Monitoring

```yaml
# synthetic/monitors.yaml
monitors:
  - name: message-send-e2e
    type: browser
    frequency: 1m
    locations: [us-east, us-west, eu-west, ap-southeast]
    script: |
      const { chromium } = require('playwright');

      (async () => {
        const browser = await chromium.launch();
        const page = await browser.newPage();

        // Login
        await page.goto('https://chat.example.com/login');
        await page.fill('#email', process.env.SYNTHETIC_USER_EMAIL);
        await page.fill('#password', process.env.SYNTHETIC_USER_PASSWORD);
        await page.click('#login-button');

        // Send test message
        await page.click('[data-testid="channel-synthetic-test"]');
        const testMsg = `Synthetic test ${Date.now()}`;
        await page.fill('[data-testid="message-input"]', testMsg);
        await page.press('[data-testid="message-input"]', 'Enter');

        // Verify message appears
        await page.waitForSelector(`text=${testMsg}`, { timeout: 5000 });

        await browser.close();
      })();

    alerts:
      - condition: failure_rate > 5%
        severity: warning
      - condition: failure_rate > 20%
        severity: critical

  - name: api-health
    type: http
    frequency: 30s
    locations: [us-east, us-west, eu-west, ap-southeast]
    requests:
      - url: https://api.chat.example.com/health
        method: GET
        assertions:
          - status == 200
          - responseTime < 500ms

      - url: https://api.chat.example.com/api/v1/channels.list
        method: GET
        headers:
          Authorization: Bearer ${SYNTHETIC_API_TOKEN}
        assertions:
          - status == 200
          - responseTime < 1000ms
```

### 11.2 Canary Analysis

```yaml
# canary/analysis.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: command-service
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: command-service

  progressDeadlineSeconds: 600

  analysis:
    # Run analysis every 1 minute
    interval: 1m
    # Max number of failed checks before rollback
    threshold: 5
    # Number of successful checks before promotion
    iterations: 10

    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m

      - name: request-duration
        thresholdRange:
          max: 100  # ms
        interval: 1m

      - name: error-rate
        thresholdRange:
          max: 1  # percent
        interval: 1m

    webhooks:
      - name: load-test
        url: http://flagger-loadtester/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://command-service-canary:8080/health"
```

---

## 12. Test Tooling

### 12.1 Tool Stack

| Category | Tool | Purpose |
|----------|------|---------|
| **Unit Testing** | Go `testing` + testify | Go service unit tests |
| **Unit Testing** | Jest | TypeScript/JavaScript tests |
| **Integration** | Testcontainers | Spin up real databases |
| **E2E** | Playwright | Browser automation |
| **Performance** | k6 | Load testing |
| **Security** | Semgrep, Trivy | SAST, container scanning |
| **Chaos** | Chaos Toolkit, Litmus | Failure injection |
| **Mocking** | gomock, WireMock | Service mocks |
| **Coverage** | go cover, Istanbul | Code coverage |
| **Reporting** | Allure, Codecov | Test reports |
| **Monitoring** | Datadog Synthetics | Production monitoring |

### 12.2 Test Infrastructure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Test Infrastructure                                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    GitHub Actions Runners                             │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │ ubuntu-     │  │ ubuntu-     │  │ macos-      │  │ self-hosted │  │   │
│  │  │ latest      │  │ latest      │  │ latest      │  │ (perf)      │  │   │
│  │  │ (unit/int)  │  │ (e2e)       │  │ (mobile)    │  │             │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Test Data Storage                                  │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │   │
│  │  │ Fixtures    │  │ Synthetic   │  │ Snapshots   │                   │   │
│  │  │ (Git)       │  │ Data (S3)   │  │ (Git LFS)   │                   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Test Reporting                                     │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │ Codecov     │  │ Allure      │  │ Grafana     │  │ Slack       │  │   │
│  │  │ (coverage)  │  │ (reports)   │  │ (perf)      │  │ (alerts)    │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Related Documents

- [Fan-Out Performance Testing](./fanout-performance-testing.md) — Detailed performance test strategy
- [Operations Guide](../operations/operations-guide.md) — Production runbooks
- [Security Architecture](../security/security-architecture.md) — Security requirements
- [CI/CD Pipeline](../operations/cicd-pipeline.md) — Deployment pipeline
