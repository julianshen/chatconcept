# Communication Platform Backend Architecture Documentation

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01
**Version:** 3.0

---

## Executive Summary

This documentation describes the architectural transformation of a communication platform backend from a monolithic Meteor.js + MongoDB Oplog architecture to a high-performance **Event-Driven Architecture (EDA)** using:

- **NATS JetStream** as the event backbone
- **Apache Cassandra** for message history storage
- **MongoDB** for metadata (users, channels, permissions)
- **Elasticsearch** for full-text search
- **Instance-level fan-out** for real-time delivery at scale

### Key Achievements

| Metric | Before | After |
|--------|--------|-------|
| Message send latency (p95) | ~150ms | < 50ms |
| Horizontal scale limit | ~8–12 instances | 50+ per service |
| Message throughput | ~5,000 msg/sec | 50,000+ msg/sec |
| Max channels per user | ~1,000 | 100,000+ |
| NATS subscriptions (system-wide) | N/A (oplog) | ~121 total |

---

## Documentation Structure

### Core Architecture

| Document | Description |
|----------|-------------|
| [Overview](./overview.md) | Executive summary, problem statement, goals, high-level architecture |
| [Detailed Design](./detailed-design.md) | Component specifications, data stores, NATS topology, key flows |

### Feature Specifications

| Document | Description |
|----------|-------------|
| [Thread Support](./features/threads.md) | Thread data model, fan-out, query endpoints |
| [Client Reconnection](./features/reconnection.md) | Tiered catchup strategy, sync protocol |
| [Read & Delivery Receipts](./features/receipts.md) | Hybrid receipt design, write amplification analysis |
| [Unread Indicators](./features/unread.md) | Read pointers, badge updates, cross-device sync |
| [Extreme Scale Partitioning](./features/extreme-scale-partitioning.md) | Fan-Out Service partitioning for >500K users |

### Diagrams

| Document | Description |
|----------|-------------|
| [C4 Diagrams](./diagrams/c4-diagrams.md) | System context, container, and component diagrams |
| [Sequence Diagrams](./diagrams/sequence-diagrams.md) | Send message, user connection, typing, thread reply, reconnection, receipts |

### Architecture Decision Records

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](./adrs/ADR-001-nats-jetstream.md) | Use NATS JetStream as Primary Event Bus | Accepted |
| [ADR-002](./adrs/ADR-002-cassandra.md) | Use Apache Cassandra for Message History | Accepted |
| [ADR-003](./adrs/ADR-003-cqrs.md) | CQRS with Async Command Processing | Accepted |
| [ADR-004](./adrs/ADR-004-mongodb-metadata.md) | Retain MongoDB for Metadata Only | Accepted |
| [ADR-005](./adrs/ADR-005-instance-level-fanout.md) | Instance-Level Fan-Out Over Per-Channel Subscriptions | Accepted |
| [ADR-006](./adrs/ADR-006-thread-storage.md) | Thread Storage — Separate Table with Dual Write | Accepted |
| [ADR-007](./adrs/ADR-007-tiered-reconnection.md) | Tiered Client Reconnection Strategy | Accepted |
| [ADR-008](./adrs/ADR-008-hybrid-receipts.md) | Hybrid Read Receipts | Accepted |
| [ADR-009](./adrs/ADR-009-fanout-partitioning.md) | Fan-Out Consistent Hash Partitioning | Proposed |

### Appendices

| Document | Description |
|----------|-------------|
| [Implementation Plan](./appendices/implementation-plan.md) | Phases, rollout strategy, timelines |
| [Risk Register](./appendices/risks.md) | Risk analysis and mitigations |
| [Open Questions](./appendices/open-questions.md) | Unresolved design questions for future consideration |

---

## Quick Start

1. **New to this architecture?** Start with [Overview](./overview.md)
2. **Need implementation details?** Read [Detailed Design](./detailed-design.md)
3. **Looking for specific features?** Check the [Features](./features/) directory
4. **Want to understand design decisions?** Review the [ADRs](./adrs/)

---

## References

- [NATS JetStream Documentation](https://docs.nats.io/nats-concepts/jetstream)
- [Apache Cassandra Documentation](https://cassandra.apache.org/doc/latest/)
- [Discord Engineering: How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)
- [CQRS Pattern — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
