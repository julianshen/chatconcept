# ADR-009: Fan-Out Consistent Hash Partitioning for Extreme Scale

**Date:** 2026-02-01
**Status:** Proposed (activate when >500K online users)
**Deciders:** Architecture Team

## Context

The replicated routing table design (each Fan-Out Worker holds a full copy) works well up to ~500K online users (~10 GB table). Beyond this, per-worker memory becomes prohibitively expensive.

| Scale | Active Channels | Online Users | Routing Table Size |
|-------|-----------------|--------------|-------------------|
| Normal (100K users) | 500K | 100K | ~2 GB |
| Large (500K users) | 2M | 500K | ~8 GB |
| **Extreme (2M+ users)** | **5M+** | **2M** | **~40 GB** |

## Decision

At extreme scale, introduce a **Fan-Out Coordinator** that routes events to partition-specific NATS subjects using **jump consistent hashing**. Partitioned Fan-Out Workers subscribe only to their assigned partition range and maintain a partial routing table.

## Rationale

| Aspect | Replicated (Current) | Partitioned (Proposed) |
|--------|---------------------|----------------------|
| Per-worker memory | Full table (~40 GB at extreme) | `table_size / num_workers` (~2.5 GB with 16 workers) |
| Message routing | Worker pulls any message, looks up locally | Coordinator hashes + publishes; worker receives only its channels |
| Adding workers | New worker loads full table (60s) | New worker loads `1/N` of table |
| Worker failure | Any surviving worker can handle any message | Only assigned partitions affected |
| Operational complexity | Low | Medium |

### Why Jump Consistent Hash?

- Produces perfectly balanced partitions (no virtual node tuning needed)
- When N changes to N+1, exactly `1/(N+1)` of keys move — minimal reshuffling
- O(ln n) computation time — negligible

### Partition Scheme

- 256 fixed partitions
- Workers assigned contiguous ranges
- Adding workers linearly reduces per-worker memory

## Consequences

**Positive:**

- Linear memory scaling with workers
- Supports millions of online users
- Zero-downtime migration from replicated to partitioned mode
- Coordinator is stateless and horizontally scalable

**Negative:**

- Additional coordinator component (2+ instances for HA)
- 5-minute event buffer needed during rebalancing
- Partition-aware failure handling is more complex than replicated model
- Should only be activated when replicated design genuinely hits memory limits
