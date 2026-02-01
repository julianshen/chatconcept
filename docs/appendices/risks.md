# Risks and Mitigations

This document catalogs identified risks and their mitigation strategies for the Communication Platform.

**Related Documents:**
- [Implementation Plan](./implementation-plan.md) — Phased rollout
- [Open Questions](./open-questions.md) — Unresolved design questions
- [ADR Index](../adrs/README.md) — Architecture decisions

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| NATS JetStream data loss (cluster failure) | Very Low | Critical | R=3 replication; regular stream snapshots; Cassandra backups are the long-term source of truth |
| Cassandra partition hotspot (viral channel) | Low | High | Daily time-bucketed partitions; further sub-partitioning for extreme channels |
| Cassandra GC pause spikes | Medium | Medium | Use G1GC/ZGC, appropriate heap sizing (8–16GB), monitor with JMX metrics, consider Cassandra 5.x with Trie-based memtables |
| Fan-Out Service routing table corruption | Low | High | Table is ephemeral and fully reconstructable from Redis + MongoDB. Kill and restart to rebuild. |
| Fan-Out Service cold start delay | Medium | Medium | 30–60s reconstruction time during which events buffer in JetStream. Pre-warm standby instances. |
| Eventual consistency confuses clients | Medium | Medium | Real-time notifications mask the gap; client library handles optimistic UI updates; REST catchup on reconnect |
| Operational complexity increase | High | Medium | Strong observability; runbooks; phased rollout allows team learning |
| WebSocket protocol migration (from DDP) | High | Medium | Versioned protocol; client library abstraction layer; support both protocols during transition |
| Webhook SSRF / abuse | Medium | High | URL allowlist/denylist validation; outbound request timeout (5s); rate limiting per webhook; HMAC signature prevents spoofing; no private/internal network access from webhook workers |
| Webhook delivery backlog during bot outage | Medium | Medium | WEBHOOKS stream has 7-day retention; auto-disable after 50 failures prevents infinite retry; backlog clears when bot recovers |

---

## Risk Categories

### Infrastructure Risks

#### NATS JetStream Cluster Failure

**Risk:** Complete loss of NATS cluster could result in event loss during the outage window.

**Likelihood:** Very Low — Requires simultaneous failure of 2+ nodes in a 3-node cluster.

**Impact:** Critical — Unprocessed events would be lost.

**Mitigations:**
1. R=3 replication ensures any 2 nodes have complete data
2. Regular stream snapshots to external storage
3. Cassandra serves as long-term source of truth for messages
4. Geo-distributed super-clusters for multi-region deployments

**Detection:**
- NATS cluster health metrics
- Replication lag alerts
- Stream sequence gap monitoring

---

#### Cassandra Partition Hotspot

**Risk:** A viral channel (millions of messages per day) could create oversized partitions, degrading read performance.

**Likelihood:** Low — Daily bucketing limits most partition sizes.

**Impact:** High — Query timeouts for affected channels.

**Mitigations:**
1. Daily time-bucketed partitions (`bucket = date(created_at)`)
2. Sub-partitioning for detected hot channels (bucket by hour)
3. Monitoring partition sizes via `nodetool tablehistograms`
4. Proactive splitting when partitions exceed 100MB

**Detection:**
- Partition size metrics
- Read latency P99 alerts per channel
- SSTable histogram analysis

---

#### Cassandra GC Pause Spikes

**Risk:** JVM garbage collection pauses could cause request timeouts and coordinator timeouts.

**Likelihood:** Medium — Common in improperly tuned deployments.

**Impact:** Medium — Temporary latency spikes, possible request failures.

**Mitigations:**
1. Use G1GC or ZGC for low-pause collection
2. Heap sizing at 8-16GB (avoid very large heaps)
3. JMX-based GC monitoring
4. Consider Cassandra 5.x with Trie-based memtables
5. Separate young gen / old gen tuning

**Detection:**
- GC pause duration metrics (> 200ms warning, > 500ms critical)
- Request latency correlation with GC events
- Heap utilization trends

---

### Application Risks

#### Fan-Out Service Routing Table Corruption

**Risk:** Bug or memory corruption could cause incorrect event routing.

**Likelihood:** Low — Straightforward data structures with well-defined operations.

**Impact:** High — Users miss messages in affected channels.

**Mitigations:**
1. Routing table is fully ephemeral and reconstructable
2. Simple restart rebuilds from Redis + MongoDB
3. Periodic consistency checks comparing table to source data
4. Multiple Fan-Out workers provide redundancy

**Detection:**
- Routing consistency health checks
- Delivery failure rate monitoring
- User-reported missing messages

**Recovery:**
```bash
# Restart affected Fan-Out worker
kubectl rollout restart deployment/fan-out-service
# Worker rebuilds routing table from Redis + MongoDB (~60s)
```

---

#### Fan-Out Service Cold Start Delay

**Risk:** 30-60 second routing table reconstruction delays real-time delivery during startup or restart.

**Likelihood:** Medium — Occurs on every restart.

**Impact:** Medium — Events buffered in JetStream, delayed but not lost.

**Mitigations:**
1. Pre-warm standby instances before production traffic
2. Rolling deployments with health checks
3. JetStream buffering provides durability during delay
4. Consider persistent routing table snapshot for faster warm-up

**Detection:**
- Startup duration metrics
- JetStream consumer lag during restarts

---

#### Eventual Consistency Client Confusion

**Risk:** Users might see inconsistent state during the async processing window.

**Likelihood:** Medium — Inherent in eventually consistent systems.

**Impact:** Medium — User confusion, duplicate submissions.

**Mitigations:**
1. Real-time WebSocket notifications deliver before DB write completes
2. Client library implements optimistic UI updates
3. REST catchup queries on reconnection
4. Clear UI indicators for pending/confirmed state
5. Idempotent operations prevent duplicate issues

**Detection:**
- User-reported inconsistency issues
- Duplicate message rate metrics

---

### Operational Risks

#### Operational Complexity Increase

**Risk:** New architecture introduces multiple new components requiring operational expertise.

**Likelihood:** High — Significant change from current architecture.

**Impact:** Medium — Slower incident response, longer learning curve.

**Mitigations:**
1. Comprehensive observability stack from day one
2. Runbooks for common operations and incidents
3. Phased rollout allows gradual team learning
4. Training sessions before production deployment
5. On-call rotation with escalation paths

**Resources Needed:**
- NATS operational training
- Cassandra administration training
- Updated monitoring dashboards
- Incident response playbooks

---

#### WebSocket Protocol Migration

**Risk:** Clients using existing DDP protocol need migration to new JSON protocol.

**Likelihood:** High — All clients must eventually migrate.

**Impact:** Medium — Client update coordination, potential compatibility issues.

**Mitigations:**
1. Versioned protocol with explicit negotiation
2. Client library abstraction layer hides protocol details
3. Support both protocols during transition period
4. Gradual client rollout with feature flags
5. Backward compatibility layer in Notification Service

**Timeline:**
- Phase 1: New protocol available, old protocol default
- Phase 2: New protocol default, old protocol supported
- Phase 3: Old protocol deprecated with warnings
- Phase 4: Old protocol removed

---

### Security Risks

#### Webhook SSRF / Abuse

**Risk:** Malicious webhook URLs could target internal services or be used for DDoS amplification.

**Likelihood:** Medium — Common attack vector for webhook systems.

**Impact:** High — Potential internal network access, reputation damage.

**Mitigations:**
1. URL allowlist/denylist validation at registration
2. Block private IP ranges (RFC 1918, loopback, link-local)
3. Outbound request timeout (5s max)
4. Rate limiting per webhook endpoint
5. HMAC signature enables recipient verification
6. Webhook workers isolated in separate network segment
7. DNS resolution validation before request

**Detection:**
- Webhook request failure patterns
- Unusual destination analysis
- Rate limit trigger alerts

---

#### Webhook Delivery Backlog

**Risk:** External bot server outage causes webhook queue to grow unbounded.

**Likelihood:** Medium — Third-party services have variable reliability.

**Impact:** Medium — Memory pressure, delayed recovery.

**Mitigations:**
1. WEBHOOKS stream has 7-day retention limit
2. Auto-disable after 50 consecutive failures
3. Exponential backoff prevents amplification
4. Webhook owner notification on auto-disable
5. Manual re-enable after bot recovery

**Monitoring:**
- Per-webhook queue depth
- Failure rate trends
- Auto-disable events

---

## Risk Assessment Matrix

```
           │ Low Impact │ Medium Impact │ High Impact │ Critical Impact
───────────┼────────────┼───────────────┼─────────────┼────────────────
Very Low   │            │               │             │ NATS cluster
Likelihood │            │               │             │ failure
───────────┼────────────┼───────────────┼─────────────┼────────────────
Low        │            │               │ Partition   │
Likelihood │            │               │ hotspot,    │
           │            │               │ Routing     │
           │            │               │ corruption  │
───────────┼────────────┼───────────────┼─────────────┼────────────────
Medium     │            │ GC pauses,    │ Webhook     │
Likelihood │            │ Cold start,   │ SSRF        │
           │            │ Consistency,  │             │
           │            │ Webhook       │             │
           │            │ backlog       │             │
───────────┼────────────┼───────────────┼─────────────┼────────────────
High       │            │ Operational   │             │
Likelihood │            │ complexity,   │             │
           │            │ Protocol      │             │
           │            │ migration     │             │
```

---

## Contingency Plans

### NATS Cluster Total Loss

1. Activate disaster recovery Cassandra reads
2. Enable REST-only mode for message delivery
3. Rebuild NATS cluster from scratch
4. Replay recent events from Cassandra audit log
5. Restore JetStream from latest snapshot

### Cassandra Cluster Degradation

1. Reduce consistency level to ONE for reads
2. Enable read caching at Query Service
3. Prioritize hot channel repairs
4. Add replacement nodes
5. Run `nodetool repair` incrementally

### Fan-Out Service Complete Failure

1. JetStream buffers all events (no loss)
2. Deploy fresh Fan-Out instances
3. Wait for routing table reconstruction (~60s)
4. Events drain from buffer automatically
5. Monitor for any missed deliveries
