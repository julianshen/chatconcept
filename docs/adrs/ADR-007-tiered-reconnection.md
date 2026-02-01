# ADR-007: Tiered Client Reconnection Strategy

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

Clients disconnect and reconnect frequently (network transitions, app backgrounding, device sleep). The reconnection protocol must sync the client's state (missed messages, unread counts) without overwhelming the server, especially for users with 100K+ channel memberships.

## Options Considered

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Always query Cassandra** | On reconnect, query every channel for new messages | Complete data, simple logic | 100K queries per reconnect = catastrophic |
| **B: JetStream replay only** | Create ephemeral consumer at client's last sequence | No DB query, fast for short gaps | Long gaps require scanning millions of messages |
| **C: Tiered strategy** | Select data source based on gap duration | Proportional cost, fast for common cases | More complex protocol |

## Decision

**Option C** — Tiered catchup strategy with 4 tiers.

## Rationale

The key insight is that reconnection cost should be **proportional to the disconnection duration**, not to the user's channel count.

| Tier | Gap Duration | Data Source | Cost |
|------|-------------|-------------|------|
| **1** | < 2 min | JetStream replay | No DB query, sub-second |
| **2** | 2 min – 1 hr | Cassandra (active channels only) | ~50 parallel queries |
| **3** | 1 hr – 24 hr | NATS KV (unread counts only) | KV reads only |
| **4** | > 24 hr | Full refresh via REST | Load on demand |

### Tier 1: Short Gap (< 2 minutes)

JetStream replay from MESSAGES stream. At 50K msg/sec globally, a 2-minute gap contains ~6M messages. JetStream can filter and replay the user's ~200–2000 relevant messages in < 1 second.

### Tier 2: Medium Gap (2 minutes – 1 hour)

Query Cassandra for the user's **active channels** (top 50 from stored session state). Skip non-active channels; compute unread counts only.

### Tier 3: Long Gap (1 hour – 24 hours)

Compute unread status per channel by comparing read pointers with channel latest. No message-level catchup. Client fetches messages via REST when user opens a channel.

### Tier 4: Very Long Gap (> 24 hours)

Full refresh via REST API. After 24 hours, incremental sync is impractical.

## Consequences

**Positive:**

- Sub-second reconnection for short gaps (most common case)
- Bounded server load regardless of user's channel count
- Graceful degradation for long offline periods

**Negative:**

- Client must implement 4 sync modes
- Server must persist `client-state` KV entries
- The `active_channels` heuristic may miss channels where important messages arrived. Mitigation: Unread sync in Tiers 3/4 covers all channels.
