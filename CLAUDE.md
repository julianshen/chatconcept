# Chat Platform Development Guide

This document defines development standards for the Chat Platform — a high-performance, event-driven messaging system built on NATS JetStream, Cassandra, and MongoDB.

---

## Project Overview

The Chat Platform migrates from a monolithic Meteor.js/MongoDB architecture to a CQRS + Event-Driven Architecture supporting:

- **50,000+ messages/second** throughput
- **100,000+ concurrent online users**
- **Users in 100K+ channels** without performance degradation
- **Sub-100ms message delivery latency (p99)**

### Technology Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.22+ |
| Event Backbone | NATS JetStream |
| Message Storage | Apache Cassandra |
| Metadata Storage | MongoDB |
| Search | Elasticsearch |
| Cache | Redis/Valkey |
| Authentication | Keycloak (OIDC) |
| API Gateway | Istio Ingress |

---

## Critical Architectural Invariants

These invariants are **non-negotiable**. Violations are treated as critical bugs.

### I1: Correctness Over Performance
> Performance optimizations must never compromise correctness. A slow correct system is preferable to a fast incorrect one.

### I2: Command Service Never Writes to Databases
> The Command Service validates and publishes events to NATS. It **never** writes directly to Cassandra, MongoDB, or any persistent store. Workers handle persistence asynchronously.

### I3: Instance-Level Fan-Out Only
> Fan-Out Service routes to **Notification Service instances**, not individual users. Per-user NATS subscriptions are forbidden.

### I4: Event Stream is Source of Truth
> NATS JetStream events are the canonical source. Databases are derived views. Any state can be reconstructed from event replay.

### I5: Never Commit Directly to Main
> All changes require a feature branch and pull request review. Direct commits to `main` are forbidden.

---

## Development Commands

### Build

```bash
# Build all services
go build ./...

# Build specific service
go build ./services/command-service
go build ./services/query-service
go build ./services/fanout-service
go build ./services/notification-service
```

### Test

```bash
# Run all tests
go test ./...

# Run with race detector
go test -race ./...

# Run with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Run specific package tests
go test -v ./services/fanout-service/...

# Run integration tests (requires testcontainers)
go test -tags=integration ./...

# Run benchmarks
go test -bench=. -benchmem ./...
```

### Lint & Format

```bash
# Format code
gofmt -w .
goimports -w .

# Run linter
golangci-lint run ./...

# Run staticcheck
staticcheck ./...

# Run govulncheck for security
govulncheck ./...
```

### Run Locally

```bash
# Start infrastructure
docker compose up -d

# Run specific service
go run ./services/command-service

# Run with hot reload (using air)
air -c .air.toml
```

---

## Code Architecture

### Service Structure

```
services/
├── command-service/     # Write path: validation + event publishing
│   ├── cmd/
│   │   └── main.go
│   ├── internal/
│   │   ├── handler/     # HTTP handlers
│   │   ├── validator/   # Input validation
│   │   ├── publisher/   # NATS publishing
│   │   └── middleware/  # Auth, rate limiting
│   └── command_test.go
├── query-service/       # Read path: REST queries
├── fanout-service/      # Event routing
├── notification-service/ # WebSocket connections
└── workers/
    ├── message-writer/  # Cassandra persistence
    ├── meta-writer/     # MongoDB persistence
    ├── search-indexer/  # Elasticsearch indexing
    └── push-worker/     # APNs/FCM delivery
```

### Package Design Principles

Following [Google Go Style Guide](https://google.github.io/styleguide/go/best-practices.html):

1. **No `util` or `helper` packages** — Names must convey purpose
2. **Cohesive packages** — If users must import multiple packages together, combine them
3. **Internal packages** — Use `internal/` for implementation details
4. **One package, one responsibility** — Each package should do one thing well

### Naming Conventions

Following [Effective Go](https://go.dev/doc/effective_go):

```go
// ✅ Good: Use package context to avoid repetition
package user

type Service struct { ... }       // Not UserService
func (s *Service) ByID(id string) // Not GetUserByID

// ✅ Good: Interfaces end in -er for single methods
type Reader interface {
    Read(p []byte) (n int, err error)
}

// ✅ Good: No "Get" prefix for getters
func (c *Channel) Name() string   // Not GetName()
func (c *Channel) SetName(n string)

// ✅ Good: MixedCaps, not underscores
var maxRetryCount int             // Not max_retry_count
type MessageWriter struct {}      // Not Message_Writer
```

### Error Handling

```go
// ✅ Good: Structured errors with context
var ErrChannelNotFound = errors.New("channel not found")

func (s *Service) GetChannel(id string) (*Channel, error) {
    ch, err := s.repo.Find(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrChannelNotFound
        }
        return nil, fmt.Errorf("find channel %s: %w", id, err)
    }
    return ch, nil
}

// ✅ Good: Check errors with errors.Is/errors.As
if errors.Is(err, ErrChannelNotFound) {
    return http.StatusNotFound
}

// ❌ Bad: String matching
if err.Error() == "channel not found" { ... }

// ✅ Good: Don't log and return - do one or the other
func process(ctx context.Context) error {
    if err := doWork(ctx); err != nil {
        return fmt.Errorf("process: %w", err)  // Return, let caller log
    }
    return nil
}
```

### Concurrency Patterns

Following Go's philosophy: **Share memory by communicating, not by sharing.**

```go
// ✅ Good: Use channels for coordination
func (fw *FanOutWorker) Start(ctx context.Context) error {
    events := make(chan Event, 100)
    errCh := make(chan error, 1)

    go func() {
        if err := fw.consume(ctx, events); err != nil {
            errCh <- err
        }
    }()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case err := <-errCh:
            return err
        case event := <-events:
            fw.route(event)
        }
    }
}

// ✅ Good: Use sync.RWMutex for read-heavy shared state
type RoutingTable struct {
    mu       sync.RWMutex
    channels map[string]*ChannelEntry
}

func (rt *RoutingTable) Get(channelID string) *ChannelEntry {
    rt.mu.RLock()
    defer rt.mu.RUnlock()
    return rt.channels[channelID]
}

func (rt *RoutingTable) Set(channelID string, entry *ChannelEntry) {
    rt.mu.Lock()
    defer rt.mu.Unlock()
    rt.channels[channelID] = entry
}

// ✅ Good: Use context for cancellation
func (w *Worker) Process(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // do work
        }
    }
}
```

---

## TDD Workflow

All development follows the **Red → Green → Refactor** cycle.

### Red Phase: Write Failing Test First

```go
// message_handler_test.go
func TestSendMessage_ValidPayload_PublishesEvent(t *testing.T) {
    // Arrange
    publisher := &MockPublisher{}
    handler := NewMessageHandler(publisher)

    payload := SendMessageRequest{
        ChannelID: "ch_general",
        Text:      "Hello, world!",
    }

    // Act
    resp, err := handler.SendMessage(context.Background(), payload)

    // Assert
    require.NoError(t, err)
    assert.Equal(t, http.StatusAccepted, resp.StatusCode)
    assert.Len(t, publisher.Published, 1)
    assert.Equal(t, "messages.send.ch_general", publisher.Published[0].Subject)
}

func TestSendMessage_EmptyText_ReturnsValidationError(t *testing.T) {
    publisher := &MockPublisher{}
    handler := NewMessageHandler(publisher)

    payload := SendMessageRequest{
        ChannelID: "ch_general",
        Text:      "",  // Invalid: empty
    }

    _, err := handler.SendMessage(context.Background(), payload)

    assert.ErrorIs(t, err, ErrEmptyMessage)
}
```

Run test — it should **fail** (red):
```bash
go test -v ./services/command-service/... -run TestSendMessage
```

### Green Phase: Write Minimal Code to Pass

```go
// message_handler.go
var ErrEmptyMessage = errors.New("message text cannot be empty")

func (h *MessageHandler) SendMessage(ctx context.Context, req SendMessageRequest) (*Response, error) {
    if req.Text == "" {
        return nil, ErrEmptyMessage
    }

    event := MessageSentEvent{
        EventID:   generateID(),
        ChannelID: req.ChannelID,
        Text:      req.Text,
        Timestamp: time.Now(),
    }

    if err := h.publisher.Publish(ctx, "messages.send."+req.ChannelID, event); err != nil {
        return nil, fmt.Errorf("publish event: %w", err)
    }

    return &Response{StatusCode: http.StatusAccepted}, nil
}
```

Run test — it should **pass** (green):
```bash
go test -v ./services/command-service/... -run TestSendMessage
```

### Refactor Phase: Improve Without Changing Behavior

```go
// Extract validation to separate function
func (h *MessageHandler) SendMessage(ctx context.Context, req SendMessageRequest) (*Response, error) {
    if err := h.validate(req); err != nil {
        return nil, err
    }

    event := h.buildEvent(req)

    if err := h.publish(ctx, req.ChannelID, event); err != nil {
        return nil, err
    }

    return &Response{StatusCode: http.StatusAccepted}, nil
}

func (h *MessageHandler) validate(req SendMessageRequest) error {
    if req.Text == "" {
        return ErrEmptyMessage
    }
    if req.ChannelID == "" {
        return ErrMissingChannelID
    }
    if len(req.Text) > MaxMessageLength {
        return ErrMessageTooLong
    }
    return nil
}
```

Run tests again — still **green**:
```bash
go test -v ./services/command-service/...
```

### TDD Checklist

Before marking a feature complete:

- [ ] All tests written **before** implementation
- [ ] Tests cover happy path and error cases
- [ ] Tests are readable and document behavior
- [ ] Code coverage ≥ 80% for new code
- [ ] No skipped tests (`t.Skip()`)
- [ ] Benchmarks added for performance-critical code

---

## Branching Strategy

### Never Commit to Main

**Invariant I5 is absolute.** All changes go through pull requests.

```bash
# ✅ Correct workflow
git checkout main
git pull origin main
git checkout -b feature/add-message-validation

# ... make changes, commit ...

git push -u origin feature/add-message-validation
gh pr create --title "Add message validation" --body "..."

# ❌ NEVER do this
git checkout main
git commit -m "quick fix"  # FORBIDDEN
git push origin main       # FORBIDDEN
```

### Branch Naming

```
feature/   — New functionality
fix/       — Bug fixes
refactor/  — Code improvements (no behavior change)
docs/      — Documentation only
test/      — Test additions/improvements
perf/      — Performance improvements
```

Examples:
```
feature/add-thread-support
fix/message-ordering-race-condition
refactor/extract-routing-table
docs/update-api-reference
test/add-fanout-benchmarks
perf/optimize-routing-lookup
```

### Pull Request Requirements

Every PR must:

1. **Pass all CI checks** (tests, lint, security scan)
2. **Have descriptive title and body**
3. **Reference related issue** (if applicable)
4. **Include tests** for new functionality
5. **Not decrease coverage** significantly
6. **Be reviewed** by at least one team member

### Small PRs

Keep PRs focused and reviewable:

- **Ideal:** 5-15 test cases, 100-300 lines changed
- **Maximum:** 500 lines (excluding generated code)
- **Split large features** into incremental PRs

---

## Commit Discipline

### Pre-Commit Checklist

Before every commit, verify:

```bash
# 1. Not on main branch
git branch --show-current  # Must NOT be "main"

# 2. All tests pass
go test -race ./...

# 3. No linter warnings
golangci-lint run ./...

# 4. Code is formatted
gofmt -l .  # Should output nothing

# 5. No security vulnerabilities
govulncheck ./...

# 6. Coverage maintained
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | grep total
```

### Commit Message Format

```
<type>: <short description>

<optional body explaining why>

<optional footer with references>
```

Types:
- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code change that neither fixes nor adds
- `test`: Adding or updating tests
- `docs`: Documentation only
- `perf`: Performance improvement
- `chore`: Build, CI, dependencies

Examples:
```
feat: add message validation for text length

Enforce 50,000 character limit on message text to prevent
storage issues and align with client-side validation.

Closes #123

---

fix: resolve race condition in routing table update

Use sync.RWMutex instead of sync.Mutex to allow concurrent
reads during presence updates.

---

test: add benchmarks for fan-out routing lookup

Measure performance with 100K, 500K, and 1M channel entries.
```

---

## Quality Standards

### Code Review Checklist

Reviewers should verify:

- [ ] **Tests first**: Tests written before implementation (TDD)
- [ ] **Coverage**: New code has ≥ 80% test coverage
- [ ] **Error handling**: Errors wrapped with context, not logged and returned
- [ ] **Concurrency**: No data races, proper synchronization
- [ ] **Naming**: Follows Go conventions, no `util` packages
- [ ] **Comments**: Explain *why*, not *what* (code explains what)
- [ ] **Performance**: No obvious inefficiencies, benchmarks for hot paths
- [ ] **Security**: No hardcoded secrets, proper input validation

### Go-Specific Standards

Following [Google Go Style Guide](https://google.github.io/styleguide/go/best-practices.html):

```go
// ✅ Good: Table-driven tests
func TestValidateMessage(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr error
    }{
        {"valid message", "hello", nil},
        {"empty message", "", ErrEmptyMessage},
        {"too long", strings.Repeat("a", 50001), ErrMessageTooLong},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateMessage(tt.input)
            if !errors.Is(err, tt.wantErr) {
                t.Errorf("got %v, want %v", err, tt.wantErr)
            }
        })
    }
}

// ✅ Good: Accept interfaces, return structs
func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}

// ✅ Good: Use context for cancellation and deadlines
func (s *Service) Process(ctx context.Context, id string) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    return s.doWork(ctx, id)
}

// ✅ Good: Defer for cleanup
func (s *Service) Process(ctx context.Context) error {
    conn, err := s.pool.Acquire(ctx)
    if err != nil {
        return fmt.Errorf("acquire connection: %w", err)
    }
    defer conn.Release()

    // use conn...
    return nil
}
```

---

## Testing Strategy

### Test Pyramid

```
              ┌─────────────────┐
              │      E2E        │  ~5% — Critical user journeys
              │    (~50 tests)  │  Playwright, real services
              └────────┬────────┘
                       │
            ┌──────────┴──────────┐
            │    Integration      │  ~15% — Service boundaries
            │    (~500 tests)     │  Testcontainers, real DBs
            └──────────┬──────────┘
                       │
    ┌──────────────────┴──────────────────┐
    │            Unit Tests               │  ~80% — Business logic
    │            (~5000 tests)            │  Fast, isolated, mocked
    └─────────────────────────────────────┘
```

### Test Organization

```go
// service_test.go — Unit tests (same package)
package service

func TestService_Process(t *testing.T) { ... }

// service_integration_test.go — Integration tests
//go:build integration

package service_test

func TestService_Integration(t *testing.T) { ... }
```

### Mocking

Use interfaces and constructor injection:

```go
// Define interface
type Publisher interface {
    Publish(ctx context.Context, subject string, data []byte) error
}

// Production implementation
type NATSPublisher struct { ... }

// Test mock
type MockPublisher struct {
    Published []PublishedMessage
    Err       error
}

func (m *MockPublisher) Publish(ctx context.Context, subject string, data []byte) error {
    if m.Err != nil {
        return m.Err
    }
    m.Published = append(m.Published, PublishedMessage{Subject: subject, Data: data})
    return nil
}
```

---

## Quality Gates

### PR Gates (Required)

| Gate | Threshold |
|------|-----------|
| Unit tests | 100% pass |
| Integration tests | 100% pass |
| Code coverage (new code) | ≥ 80% |
| Code coverage (total) | No decrease |
| Linter (golangci-lint) | 0 errors |
| Security (govulncheck) | 0 critical/high |
| Build | Succeeds |

### Release Gates (Required)

| Gate | Threshold |
|------|-----------|
| All PR gates | Pass |
| E2E tests | 100% pass |
| Performance baseline | No regression > 10% |
| Security audit | 0 critical, 0 high |
| Staging smoke tests | Pass |

---

## Performance Targets

| Metric | Target | Validation |
|--------|--------|------------|
| Message send latency (p99) | < 50ms | ⬜ Pending |
| Message delivery latency (p99) | < 100ms | ⬜ Pending |
| Fan-out routing lookup | < 1ms | ⬜ Pending |
| JetStream consumer lag | < 1,000 msgs | ⬜ Pending |
| Throughput (sustained) | 50,000 msg/sec | ⬜ Pending |
| Memory (Fan-Out routing table) | < 3GB | ⬜ Pending |

### Benchmark Requirements

Performance-critical code must have benchmarks:

```go
func BenchmarkRoutingTableLookup(b *testing.B) {
    rt := NewRoutingTable()
    // Pre-populate with 500K channels
    for i := 0; i < 500000; i++ {
        rt.Set(fmt.Sprintf("ch_%d", i), &ChannelEntry{})
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        rt.Get("ch_250000")
    }
}
```

Run benchmarks:
```bash
go test -bench=BenchmarkRoutingTableLookup -benchmem ./services/fanout-service/...
```

---

## Workflow Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Development Workflow                                 │
│                                                                              │
│  1. CREATE BRANCH                                                            │
│     git checkout -b feature/add-message-validation                          │
│                                                                              │
│  2. TDD CYCLE (repeat)                                                       │
│     ┌─────────┐      ┌─────────┐      ┌─────────────┐                       │
│     │   RED   │ ───► │  GREEN  │ ───► │  REFACTOR   │                       │
│     │  Write  │      │  Make   │      │  Improve    │                       │
│     │  Test   │      │  Pass   │      │  Code       │                       │
│     └─────────┘      └─────────┘      └─────────────┘                       │
│                                              │                               │
│                                              ▼                               │
│  3. VERIFY QUALITY                                                           │
│     go test -race ./...                                                     │
│     golangci-lint run ./...                                                 │
│     govulncheck ./...                                                       │
│                                                                              │
│  4. COMMIT                                                                   │
│     git add .                                                               │
│     git commit -m "feat: add message validation"                            │
│                                                                              │
│  5. PUSH & CREATE PR                                                         │
│     git push -u origin feature/add-message-validation                       │
│     gh pr create --title "Add message validation"                           │
│                                                                              │
│  6. CODE REVIEW                                                              │
│     Address feedback, push updates                                          │
│                                                                              │
│  7. MERGE (after approval)                                                   │
│     gh pr merge --squash                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## References

- [Google Go Style Guide](https://google.github.io/styleguide/go/best-practices.html)
- [Effective Go](https://go.dev/doc/effective_go)
- [Architecture Overview](./docs/overview.md)
- [Detailed Design](./docs/detailed-design.md)
- [Testing Strategy](./docs/testing/testing-strategy.md)
- [Fan-Out Performance Testing](./docs/testing/fanout-performance-testing.md)

---

**Remember: Quality is not negotiable. Invariant I5 (never commit to main) is as critical as a security vulnerability.**
