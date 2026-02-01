# ADR-010: Multi-Region Deployment with NATS Super-Clusters

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

The platform targets ~50K users distributed globally across multiple geographic regions (US-East, US-West, EU, Asia). Channel members can be from different regions, causing cross-region latency to degrade real-time messaging experience. The goal is to reduce delivery latency for same-region users (<50ms) while keeping cross-region latency acceptable (<300ms).

Key questions:
1. How should NATS clusters be deployed across regions?
2. Should the Fan-Out Service be regionalized or remain global?
3. How should data stores handle multi-region topology?

## Decision

Adopt a **Regional Edge + Global Backbone** architecture:

1. **NATS Super-Cluster with Gateway Links:** Each region runs its own NATS cluster (3 nodes) connected via gateway links for cross-region event propagation.

2. **Global Fan-Out Service:** Keep the Fan-Out Service in the primary region (US-East) rather than regionalizing it.

3. **Regional Edge Services:** Deploy Notification Service and Query Service in each region for low-latency client connections.

4. **Multi-DC Data Stores:** Cassandra with multi-DC replication, Redis with read replicas, MongoDB with secondaries in each region.

## Rationale

### NATS Super-Cluster vs Single Global Cluster

| Approach | Same-Region Latency | Cross-Region Latency | Complexity |
|----------|---------------------|----------------------|------------|
| Single global cluster | ~150ms (all traffic crosses regions) | ~150ms | Low |
| **Super-cluster with gateways** | **<50ms** | **<300ms** | Medium |
| Fully independent clusters | <50ms | No cross-region delivery | High (data sync) |

NATS super-clusters provide **interest-based forwarding**: only events with subscribers in remote regions traverse gateways. A message in a US-only channel never crosses the Atlantic.

### Global vs Regional Fan-Out Service

At 50K users, the routing table is ~2GB — well within single-instance memory capacity.

| Approach | Pros | Cons |
|----------|------|------|
| **Global Fan-Out (chosen)** | Simple routing logic, single source of truth, no cross-region coordination | Cross-region hop for remote notifications |
| Regional Fan-Out | Lower latency for local events | Requires routing table synchronization across regions, split-brain risks, 3x memory usage |

The cross-region latency penalty (~100-150ms) is acceptable given the complexity savings. Regional Fan-Out would require distributed consensus for the routing table, adding operational risk.

### Data Store Topology

**Cassandra:**
- Multi-DC deployment with async replication
- Writes go to primary DC (US-East) with LOCAL_QUORUM
- Reads use LOCAL_ONE from regional replicas
- ~100-200ms replication lag is acceptable for historical data

**Redis/Valkey:**
- Primary in US-East handles all writes
- Read replicas in each region for presence/typing queries
- Async replication acceptable for ephemeral state

**MongoDB:**
- Primary-Secondary topology
- Metadata changes are infrequent
- Regional Query Services read from nearest secondary

## Architecture Overview

```
                ┌─────────────────────────────────────┐
                │      NATS Super-Cluster (Global)    │
                │  ┌─────────┐       ┌─────────┐      │
                │  │US-East  │◄─────►│US-West  │      │
                │  │ Cluster │       │ Cluster │      │
                │  └────┬────┘       └────┬────┘      │
                │       │   Gateway Links  │          │
                │  ┌────┴────┐       ┌────┴────┐      │
                │  │  EU     │◄─────►│  Asia   │      │
                │  │ Cluster │       │ Cluster │      │
                │  └─────────┘       └─────────┘      │
                └─────────────────────────────────────┘
                                │
    ┌───────────────────────────┼───────────────────────────┐
    │                           │                           │
    ▼                           ▼                           ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│ Regional Edge │      │ Regional Edge │      │ Regional Edge │
│   US-East     │      │      EU       │      │     Asia      │
│   (Primary)   │      │               │      │               │
│               │      │               │      │               │
│ • Fan-Out Svc │      │ • Notif Svc   │      │ • Notif Svc   │
│ • Notif Svc   │      │ • Query Svc   │      │ • Query Svc   │
│ • Query Svc   │      │ • Redis Read  │      │ • Redis Read  │
│ • Cmd Svc     │      │ • Cassandra   │      │ • Cassandra   │
│ • Workers     │      │   (LOCAL_ONE) │      │   (LOCAL_ONE) │
│ • Redis Pri   │      │ • Mongo Sec   │      │ • Mongo Sec   │
│ • Cassandra   │      │               │      │               │
│ • Mongo Pri   │      │               │      │               │
└───────────────┘      └───────────────┘      └───────────────┘
```

## Latency Analysis

| Scenario | Path | Expected Latency |
|----------|------|------------------|
| Same-region message | User→Regional NATS→Fan-Out→Regional NATS→Regional Notif→User | <50ms |
| Cross-region message | User→Regional NATS→Gateway→US-East NATS→Fan-Out→Gateway→Regional NATS→Regional Notif→User | <300ms |
| History query (any region) | User→Regional Query→Regional Cassandra | <30ms |
| Presence update (same region) | User→Primary Redis→Replica | <200ms propagation |

## Consequences

**Positive:**

- Same-region message delivery under 50ms
- Cross-region delivery under 300ms (acceptable for chat)
- Regional Query Services provide low-latency reads globally
- GeoDNS/Anycast routes users to nearest Notification Service
- Cassandra's multi-DC replication is battle-tested
- Operational simplicity: single Fan-Out Service, no distributed routing table

**Negative:**

- Primary region (US-East) is a single point of failure for writes
- **Isolated regions cannot send/receive messages** — all writes require primary region connectivity
- Cross-region events incur gateway hop latency
- Redis write amplification (all presence updates go to primary)
- Regional infrastructure cost (3x clusters, replicas)

**Mitigations:**

- Automated failover for NATS cluster leader election
- JetStream provides durability during brief gateway outages
- Consider multi-primary Redis (Valkey) for presence in future
- Start with 2 regions (US-East + EU), add Asia based on user distribution

## Future Consideration: Regional Autonomy

The current design prioritizes **consistency over availability**. If regional autonomy during network partitions becomes a requirement, the architecture can evolve to support **same-region chat during isolation**:

### Required Changes

| Component | Current | Regional Autonomy |
|-----------|---------|-------------------|
| Command Service | Primary region only | Deploy in each region |
| Fan-Out Service | Primary region only | Deploy in each region |
| NATS Streams | Global streams | Regional streams (`MESSAGES_{region}`) with cross-region mirroring |
| Message IDs | Centralized | Region-prefixed (`msg_us_...`, `msg_eu_...`) |
| Routing Table | Global (~2GB) | Regional subset (~900MB) + remote user index |

### Recovery Mechanism

When connectivity restores after a partition:

1. **NATS mirrors sync automatically** — JetStream mirrors catch up with source streams
2. **Fan-Out Services process missed events** — Each region processes events from other regions' mirrors
3. **Clients see interleaved messages** — Ordered by timestamp, not arrival order
4. **Presence state reconciled** — Regional Fan-Out Services exchange online user lists

### Trade-off

| Aspect | Current Design | With Regional Autonomy |
|--------|----------------|------------------------|
| Same-region chat during partition | :x: No | :white_check_mark: Yes |
| Message ordering | Strict global | Eventually consistent |
| Conflict resolution | Not needed | Last-Write-Wins |
| Complexity | Lower | Higher |

**Recommendation:** Implement regional autonomy only if:
- >30% of users are in non-primary regions
- Network partitions are not rare
- Availability is prioritized over strict consistency

See [Multi-Region Feature Documentation](../features/multi-region.md#10-future-enhancement-regional-autonomy) for detailed design.

## Related ADRs

- [ADR-001: Use NATS JetStream as the Primary Event Bus](./ADR-001-nats-jetstream.md)
- [ADR-005: Instance-Level Fan-Out Architecture](./ADR-005-instance-level-fanout.md)
