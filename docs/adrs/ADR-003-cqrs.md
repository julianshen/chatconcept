# ADR-003: CQRS with Async Command Processing

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

The current Meteor.js architecture processes commands synchronously: a client sends a message, the server writes to MongoDB, waits for the write to succeed, then notifies clients via oplog. This couples the client's response latency to the database write latency.

## Decision

Adopt **CQRS (Command Query Responsibility Segregation)** with asynchronous command processing.

The Command Service validates and publishes events to NATS JetStream, returning `HTTP 202 Accepted` immediately. Database persistence happens asynchronously via worker consumers.

## Rationale

### Benefits of CQRS + Async Processing

1. **Decoupled latency:** Client response time is bounded by NATS publish latency (~1–5ms), not database write latency (~10–50ms+).

2. **Independent scaling:** Write workers, read services, and notification services scale independently based on their specific resource requirements.

3. **Resilience:** If Cassandra is temporarily unavailable, messages are buffered in NATS JetStream (30-day retention). Workers resume processing when the database recovers. No messages are lost.

4. **Polyglot persistence:** Different read models can be built from the same event stream (Cassandra for history, Elasticsearch for search, analytics DB for metrics) without changing the write path.

### Command Flow

```
Client → Gateway → Command Service → NATS JetStream → Workers → Databases
                              ↓
                    HTTP 202 Accepted
                    (client continues)
```

## Consequences

**Positive:**

- Client-perceived latency drops significantly
- System survives database outages without data loss
- New read models can be added by creating new consumers — zero changes to the write path

**Negative:**

- **Eventual consistency:** A brief window where reads may not reflect the latest write. Mitigation: Real-time notifications deliver the message to clients immediately via WebSocket, so the UI is consistent even if the database lags.

- **Increased system complexity:** More moving parts to monitor and debug. Mitigation: Strong observability stack (OpenTelemetry, consumer lag monitoring, correlation IDs).

- **Client error handling:** `202 Accepted` means the command was _accepted for processing_, not that it _succeeded_. If validation passes but persistence fails (rare), the client won't know immediately. Mitigation: Dead-letter queue for failed events; admin monitoring dashboard; client can verify via subsequent GET.
