# ADR-005: Instance-Level Fan-Out Over Per-Channel Subscriptions

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

Users may belong to hundreds of thousands of channels. The traditional approach — one NATS subscription per channel per user — creates a combinatorial explosion that exceeds the practical limits of any message broker.

At 100K concurrent users × 100K channels per user, the naive model requires up to **10 billion subscription evaluations**.

## Decision

Adopt an **instance-level fan-out architecture** where a dedicated Fan-Out Service routes events to Notification Service instance inboxes rather than individual user subscriptions.

Each Notification Service instance subscribes to exactly **one** NATS subject — its own instance inbox.

## Rationale

| Aspect | Per-Channel Subscriptions (Naive) | Instance-Level Fan-Out (Chosen) |
|--------|----------------------------------|--------------------------------|
| **NATS subscriptions per instance** | O(users × channels) — billions | O(1) — exactly 1 per instance |
| **NATS publishes per message** | 1 (NATS evaluates internally) | O(instances with members) — 1–100 |
| **NATS server memory** | >1GB per million subscriptions | Negligible (~121 total) |
| **Subscription management on connect** | Create 100K+ subscriptions | Zero NATS subscriptions |
| **User connect latency** | O(channels) NATS subscribe calls | O(channels) in-memory map inserts (~5ms) |
| **Complexity** | Simple per-service, system bottleneck | Fan-Out Service is additional component |

### How It Works

1. **Event Ingestion:** Command Service publishes a single event to NATS JetStream
2. **Instance-Level Routing:** Fan-Out Service looks up routing table: "Which instances have online users in this channel?" Publishes to each instance's inbox.
3. **Local Delivery:** Notification Service receives on its single inbox subscription and routes locally to WebSocket connections.

### Memory Trade-off

The Fan-Out Service maintains an in-memory routing table (~2GB at 100K online users). This fits comfortably in a modern server's memory and is fully reconstructable from Redis + MongoDB.

## Consequences

**Positive:**

- System supports users in 100K+ channels with zero degradation
- NATS server memory usage remains constant regardless of user channel count
- Adding Notification Service instances requires zero NATS reconfiguration
- Fan-Out Service is stateless (routing table is reconstructable)

**Negative:**

- Fan-Out Service is an additional component that must be operated, monitored, and scaled
- Single extra network hop for event delivery. Adds ~1–3ms latency. Acceptable for chat.
- Routing table cold start takes 30–60 seconds. During this period, events are delayed but not lost (buffered in JetStream).
- All real-time event delivery depends on the Fan-Out Service being operational. Mitigation: Multiple workers share the load; JetStream buffers events during brief outages.
