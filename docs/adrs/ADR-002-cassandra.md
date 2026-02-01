# ADR-002: Use Apache Cassandra for Message History Storage

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

Message history is the highest-volume data in a communication platform. The access pattern is predominantly:
1. Append-only writes (high volume)
2. Time-ordered range reads (load recent messages for a channel)
3. Occasional point reads (fetch specific message by ID)

MongoDB's B-tree based storage engine is not optimized for this write-heavy, time-series-like workload.

## Decision

Use **Apache Cassandra** as the dedicated message history database.

## Rationale

| Requirement | Apache Cassandra | PostgreSQL | MongoDB (Current) |
|-------------|------------------|------------|-------------------|
| **Write throughput** | Excellent — LSM-tree, append-only | Good — B-tree, WAL | Good — WiredTiger B-tree |
| **Linear horizontal scaling** | ✅ Add nodes, vnode-based auto-rebalancing | ⚠️ Requires manual partitioning | ⚠️ Sharding is complex |
| **Time-range queries** | ✅ Clustering key + TWCS compaction | ✅ B-tree indexes work | ⚠️ Compound indexes |
| **Partition design** | ✅ Explicit partition key control | N/A | ⚠️ Shard key is hard to change |
| **Compaction** | ✅ TWCS eliminates write amplification | N/A (VACUUM) | N/A (WiredTiger) |
| **Operational maturity** | ✅ Battle-tested (Apple, Netflix, Discord) | Medium | Medium |

### Why Cassandra vs. ScyllaDB?

Both are CQL-compatible. ScyllaDB offers higher per-node performance. However, we choose Cassandra for:

- **Operational maturity:** Battle-tested at extreme scale for over a decade
- **Community and ecosystem:** Largest community, mature tooling (Medusa, Reaper)
- **Talent availability:** More widely available than ScyllaDB specialists
- **Cost model:** Lower per-node throughput compensated by adding more (cheaper) nodes

### Why not PostgreSQL?

PostgreSQL is excellent for metadata, but at message volumes exceeding 10,000+ writes/sec with multi-year retention, its B-tree index maintenance, VACUUM overhead, and single-primary write bottleneck become limiting.

### Why not keep MongoDB for messages?

MongoDB is retained for metadata (flexible schema, rich queries), but message history benefits from Cassandra's write-optimized LSM architecture, explicit partition control, and linear horizontal scaling.

## Consequences

**Positive:**

- Write throughput scales linearly with cluster size
- Time-range queries optimally served by clustering key ordering + TWCS
- No single point of failure (masterless, peer-to-peer architecture)
- Battle-tested at extreme scale with well-understood failure modes

**Negative:**

- Adds a new database technology. Mitigation: Extensive documentation and community support.
- JVM-based: requires careful GC tuning. Mitigation: Use G1GC/ZGC with appropriate heap sizing.
- Limited ad-hoc query capability. Mitigation: Complex queries served by Elasticsearch.
- No multi-document transactions. Mitigation: Message operations are single-partition writes.
