# ADR-001: Use NATS JetStream as the Primary Event Bus

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

The current architecture uses MongoDB Oplog Tailing as a de facto event bus. We need a dedicated message broker that supports:
- Persistent event streaming
- Consumer groups for horizontal scaling
- Lightweight ephemeral pub/sub for real-time notifications

## Decision

Use **NATS JetStream** as the primary event bus and message persistence layer.

## Rationale

| Requirement | NATS JetStream | Apache Kafka | RabbitMQ |
|-------------|----------------|--------------|----------|
| **Persistent streaming** | ✅ Streams with configurable retention, R=N replication via RAFT | ✅ Log-based, partition replication | ⚠️ Requires plugins |
| **Consumer groups** | ✅ Pull consumers share work automatically; no partitioning required | ✅ But requires partition management | ✅ Competing consumers |
| **Lightweight pub/sub** | ✅ Core NATS — zero overhead, no persistence | ❌ All topics are persistent | ⚠️ More overhead |
| **KV Store** | ✅ Built-in with TTL, watchers, history | ❌ Requires external system | ❌ Requires external system |
| **Exactly-once delivery** | ✅ Publisher dedup + double-ack | ✅ Idempotent producer + transactions | ⚠️ At-least-once |
| **Operational complexity** | Low — single binary | High — ZooKeeper/KRaft | Medium — Erlang runtime |
| **Scaling without partitions** | ✅ Pull consumers scale horizontally | ❌ Requires repartitioning | ✅ But queue-based |
| **Message replay** | ✅ DeliverAll, DeliverByStartSequence, DeliverByStartTime | ✅ Offset-based replay | ❌ Messages consumed and gone |
| **Latency** | Sub-millisecond publish | Low-millisecond | Low-millisecond |

### Key Differentiators

1. **No partition management:** Unlike Kafka, NATS pull consumers scale horizontally without partition configuration. Adding a new worker is as simple as binding to the same durable consumer name.

2. **Dual-mode messaging:** NATS provides both persistent streaming (JetStream) and ephemeral pub/sub (Core NATS) in the same system. Typing indicators use ephemeral pub/sub with zero persistence overhead.

3. **Built-in KV Store:** The NATS KV Store provides a distributed key-value store with TTL and watchers — eliminating the need for a separate Redis deployment.

4. **Operational simplicity:** NATS is a single Go binary. A 3-node JetStream cluster is straightforward compared to Kafka's multi-component ecosystem.

## Consequences

**Positive:**

- Dramatically simplified operational footprint
- Seamless horizontal scaling of workers without partition rebalancing
- Sub-millisecond publish latency for the command path
- Built-in event replay for recovery and debugging
- Core NATS pub/sub enables the instance-inbox fan-out pattern

**Negative:**

- NATS JetStream is less battle-tested at extreme scale (>1M messages/sec sustained) compared to Kafka. Mitigation: NATS super-clusters and careful stream design.
- Smaller ecosystem of connectors and tooling. Mitigation: Our workers are custom-built anyway.
- No built-in schema registry. Mitigation: Use lightweight JSON schema validation in the Command Service.
