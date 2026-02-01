# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) documenting key technical choices for the Communication Platform Backend.

## Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [ADR-001](./ADR-001-nats-jetstream.md) | Use NATS JetStream as Primary Event Bus | Accepted | 2026-02-01 |
| [ADR-002](./ADR-002-cassandra.md) | Use Apache Cassandra for Message History | Accepted | 2026-02-01 |
| [ADR-003](./ADR-003-cqrs.md) | CQRS with Async Command Processing | Accepted | 2026-02-01 |
| [ADR-004](./ADR-004-mongodb-metadata.md) | Retain MongoDB for Metadata Only | Accepted | 2026-02-01 |
| [ADR-005](./ADR-005-instance-level-fanout.md) | Instance-Level Fan-Out Over Per-Channel Subscriptions | Accepted | 2026-02-01 |
| [ADR-006](./ADR-006-thread-storage.md) | Thread Storage — Separate Table with Dual Write | Accepted | 2026-02-01 |
| [ADR-007](./ADR-007-tiered-reconnection.md) | Tiered Client Reconnection Strategy | Accepted | 2026-02-01 |
| [ADR-008](./ADR-008-hybrid-receipts.md) | Hybrid Read Receipts | Accepted | 2026-02-01 |
| [ADR-009](./ADR-009-fanout-partitioning.md) | Fan-Out Consistent Hash Partitioning | Proposed | 2026-02-01 |

## ADR Template

When creating a new ADR, use the following template:

```markdown
# ADR-XXX: [Title]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded
**Deciders:** [Names or team]

## Context

[Describe the issue motivating this decision]

## Decision

[Describe the decision that was made]

## Rationale

[Explain why this decision was made, including alternatives considered]

## Consequences

**Positive:**
- [List positive consequences]

**Negative:**
- [List negative consequences and mitigations]
```

## Categories

### Core Infrastructure
- ADR-001: Event Bus (NATS JetStream)
- ADR-002: Message Storage (Cassandra)
- ADR-004: Metadata Storage (MongoDB)

### Architecture Patterns
- ADR-003: CQRS + Async Processing
- ADR-005: Instance-Level Fan-Out

### Feature Design
- ADR-006: Thread Storage
- ADR-007: Client Reconnection
- ADR-008: Read Receipts

### Scaling
- ADR-009: Fan-Out Partitioning
