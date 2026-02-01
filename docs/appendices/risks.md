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
| Cross-region network partition | Low | High | NATS gateway link failure isolates regions; messages buffer until connectivity restored; GeoDNS failover to alternate region |
| Regional NATS cluster failure | Low | High | Users in affected region cannot receive real-time updates; GeoDNS redirects to next-nearest region; automatic recovery via RAFT |
| Cassandra cross-DC replication lag | Medium | Low | Regional reads may serve stale data (100-200ms lag); acceptable for message history; real-time updates via WebSocket are authoritative |
| Primary region complete failure | Very Low | Critical | All writes fail; manual failover to secondary region required (15-30min RTO); regional read replicas continue serving queries |
| Event reconciliation conflicts (Regional Autonomy) | Medium | Medium | Same message edited in multiple regions during partition; Last-Write-Wins resolution may surprise users; clear conflict indicators in UI |
| Split-brain during partition recovery (Regional Autonomy) | Low | High | Brief window where regions disagree on state; NATS mirror sync and presence reconciliation resolve within minutes |

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

---

## Cross-Region Risks

### Cross-Region Network Partition (Gateway Link Failure)

**Risk:** Network issues between regions sever NATS gateway links, preventing cross-region message delivery.

**Likelihood:** Low — Requires failure of both primary and backup network paths.

**Impact:** High — Users in isolated region miss messages from other regions.

**Mitigations:**
1. NATS buffers messages for disconnected regions
2. Multiple gateway endpoints per region for redundancy
3. GeoDNS can redirect users to alternate region if partition persists
4. JetStream provides durability during brief outages
5. Monitoring on gateway link latency and throughput

**Detection:**
- Gateway link heartbeat failures
- Consumer lag spike for cross-region Notification Service instances
- Cross-region message delivery latency alerts

**Recovery:**
1. Network path automatically restores (typically <5 minutes)
2. Buffered messages drain to affected region
3. No manual intervention required for transient partitions
4. For prolonged partitions, redirect users via DNS failover

---

### Regional NATS Cluster Failure

**Risk:** All 3 NATS nodes in a regional cluster fail simultaneously.

**Likelihood:** Low — Requires multiple simultaneous failures.

**Impact:** High — Users connected to that region cannot receive real-time updates.

**Mitigations:**
1. RAFT-based leader election within cluster
2. Placement across multiple availability zones
3. GeoDNS health checks redirect to healthy regions
4. Users can manually reconnect to alternate region
5. Primary region NATS continues operating independently

**Detection:**
- NATS cluster health metrics
- Regional Notification Service connection failures
- User connection drop alerts

**Recovery:**
1. GeoDNS/LB shifts WebSocket traffic to next-nearest region
2. Users reconnect with increased latency
3. Failed region NATS cluster recovers via RAFT
4. Traffic gradually returns when health checks pass

---

### Cassandra Cross-DC Replication Lag

**Risk:** Async replication between DCs creates read inconsistency window.

**Likelihood:** Medium — Inherent in async replication design.

**Impact:** Low — Users may see slightly stale message history.

**Mitigations:**
1. Real-time WebSocket notifications are authoritative (not affected)
2. 100-200ms lag is acceptable for historical data
3. Regional Query Services use LOCAL_ONE for low latency
4. Monitor replication lag metrics
5. Can temporarily switch to cross-DC consistency if needed

**Acceptable Because:**
- WebSocket delivers new messages in real-time before DB replication
- Message history queries tolerate brief staleness
- Users refresh naturally as they scroll

---

### Primary Region Complete Failure

**Risk:** US-East region (primary) suffers complete outage affecting Command Service, Fan-Out Service, and primary data stores.

**Likelihood:** Very Low — Requires catastrophic regional failure.

**Impact:** Critical — All write operations fail; only regional reads continue.

**Mitigations:**
1. Regional read replicas continue serving queries
2. Regional Notification Services maintain existing WebSocket connections
3. Documented manual failover procedure
4. Cassandra can promote secondary DC to primary
5. MongoDB can elect new primary from secondaries

**Manual Failover Procedure:**
1. Verify primary region is unrecoverable (15-minute observation)
2. Promote EU-West Cassandra DC to primary
3. MongoDB election happens automatically (may need manual intervention)
4. Deploy Command Service and Fan-Out Service to EU-West
5. Update NATS gateway configuration
6. DNS cutover to EU-West as new primary
7. Communicate to users about potential data inconsistency window

**RTO:** 15-30 minutes for manual failover

**RPO:** Potential loss of in-flight events not yet replicated to secondary DCs

---

### Event Reconciliation Conflicts (Regional Autonomy)

**Risk:** During a network partition, the same message could be edited or deleted in multiple regions, leading to conflicting states.

**Likelihood:** Medium — Increases with partition duration and channel activity.

**Impact:** Medium — Users may see unexpected edit results; no data loss.

**Mitigations:**
1. Last-Write-Wins (LWW) resolution by timestamp
2. UI indicators showing "edited during partition" or conflict marker
3. Audit log preserves all versions for manual review if needed
4. Deterministic resolution ensures all regions converge to same state
5. Consider CRDTs for future enhancement if LWW is insufficient

**Example Scenario:**
```
Message msg_us_001 during 30-minute partition:
- US (10:05:00): Edit to "Hello world!"
- EU (10:05:30): Edit to "Hello everyone!"
Result: "Hello everyone!" wins (later timestamp)
```

---

### Split-Brain During Partition Recovery (Regional Autonomy)

**Risk:** Brief window after partition recovery where regions have inconsistent views of presence, messages, and channel membership.

**Likelihood:** Low — Only occurs during recovery window.

**Impact:** High — Users may temporarily see inconsistent state.

**Mitigations:**
1. NATS stream mirroring syncs automatically on reconnection
2. Presence reconciliation protocol exchanges user lists
3. Convergence typically within 1-5 minutes
4. Client SDK handles out-of-order message delivery gracefully
5. Clear "syncing" indicator in UI during reconciliation

**Recovery Timeline:**
```
T=0:        Partition ends, gateway links reconnect
T=0-30s:    NATS detects reconnection, starts mirror sync
T=30s-2m:   Message streams synchronize (depending on volume)
T=1-3m:     Presence reconciliation completes
T=3-5m:     Full consistency restored
```

**Detection:**
- Mirror lag metrics spike then decrease
- Presence sync messages between regions
- Client reconnection rate normalization

---

## Updated Risk Assessment Matrix

```
           │ Low Impact │ Medium Impact │ High Impact │ Critical Impact
───────────┼────────────┼───────────────┼─────────────┼────────────────
Very Low   │            │               │             │ NATS cluster
Likelihood │            │               │             │ failure,
           │            │               │             │ Primary region
           │            │               │             │ failure
───────────┼────────────┼───────────────┼─────────────┼────────────────
Low        │            │               │ Partition   │
Likelihood │            │               │ hotspot,    │
           │            │               │ Routing     │
           │            │               │ corruption, │
           │            │               │ Network     │
           │            │               │ partition,  │
           │            │               │ Regional    │
           │            │               │ NATS fail,  │
           │            │               │ Split-brain*│
───────────┼────────────┼───────────────┼─────────────┼────────────────
Medium     │ Cassandra  │ GC pauses,    │ Webhook     │
Likelihood │ replication│ Cold start,   │ SSRF        │
           │ lag        │ Consistency,  │             │
           │            │ Webhook       │             │
           │            │ backlog,      │             │
           │            │ Reconciliation│             │
           │            │ conflicts*    │             │
───────────┼────────────┼───────────────┼─────────────┼────────────────
High       │            │ Operational   │             │
Likelihood │            │ complexity,   │             │
           │            │ Protocol      │             │
           │            │ migration     │             │

* Regional Autonomy risks (Phase 8) — only applicable if implemented
```
