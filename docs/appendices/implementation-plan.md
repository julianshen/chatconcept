# Implementation Plan

This document outlines the phased implementation approach for the Communication Platform.

**Related Documents:**
- [Architecture Overview](../overview.md) — High-level architecture
- [Risks and Mitigations](./risks.md) — Risk register
- [ADR Index](../adrs/README.md) — Architecture decisions

---

## Implementation Phases

| Phase | Scope | Duration (Est.) | Dependencies | Risk |
|-------|-------|-----------------|--------------|------|
| **0: Foundation** | Deploy NATS JetStream cluster (3-node). Deploy Cassandra cluster (3-node, RF=3). Set up CI/CD, observability stack. | 2–3 weeks | Infrastructure provisioning | Low |
| **1: Command Path** | Build Command Service. Implement event publishing for `messages.send`, `messages.edit`, `messages.delete`. Build Message Writer worker. Validate Cassandra schema and write path. | 3–4 weeks | Phase 0 | Medium |
| **2: Query Path** | Build Query Service implementing Rocket.Chat REST API read endpoints. Connect to Cassandra for message history, MongoDB for metadata. | 2–3 weeks | Phase 1 | Low |
| **3: Fan-Out & Notification** | Build Fan-Out Service with routing table, presence watcher, membership consumer. Build Notification Service with WebSocket protocol, local routing, instance inbox subscription. | 4–5 weeks | Phase 1 | **High** (core innovation) |
| **4: Workers & Webhooks** | Build Search Indexer, Push Notification Worker, Meta Writer. Build Webhook Dispatcher with outgoing delivery, retry, HMAC signing, delivery log, auto-disable. Build incoming webhook endpoint in Command Service. Set up Elasticsearch. Set up WEBHOOKS stream and webhook Cassandra tables. | 3–4 weeks | Phase 1 | Medium |
| **5: Migration & Cutover** | Data migration from MongoDB messages to Cassandra. Dual-write period. Shadow traffic testing. Gradual client cutover. Migrate existing Rocket.Chat integrations to incoming/outgoing webhook model. | 3–4 weeks | Phases 1–4 | High |
| **6: Decommission** | Remove Meteor.js oplog tailing. Remove message collections from MongoDB. Optimize and tune. | 1–2 weeks | Phase 5 stable | Low |
| **7: Multi-Region** | Deploy NATS super-cluster with gateway links. Set up regional Cassandra DCs, Redis replicas, MongoDB secondaries. Deploy regional Notification and Query Services. Configure GeoDNS routing. | 4–6 weeks | Phase 6 stable | High |
| **8: Regional Autonomy** *(Optional)* | Deploy regional Command and Fan-Out Services. Implement regional NATS streams with cross-region mirroring. Add event reconciliation and conflict resolution. | 4–6 weeks | Phase 7 stable | Very High |

---

## Phase Details

### Phase 0: Foundation (2-3 weeks)

**Objectives:**
- Production-ready NATS JetStream cluster
- Production-ready Cassandra cluster
- Observability infrastructure

**Deliverables:**
- [ ] NATS JetStream 3-node cluster deployed
- [ ] Cassandra 3-node cluster with RF=3 deployed
- [ ] CI/CD pipelines configured
- [ ] Prometheus + Grafana monitoring stack
- [ ] OpenTelemetry tracing infrastructure
- [ ] Centralized logging (ELK or Loki)

**Success Criteria:**
- NATS cluster survives single-node failure
- Cassandra cluster survives single-node failure
- All metrics dashboards operational

---

### Phase 1: Command Path (3-4 weeks)

**Objectives:**
- Validate CQRS write path
- Establish event schema patterns
- Prove Cassandra write performance

**Deliverables:**
- [ ] Command Service with message endpoints
- [ ] Message Writer pull consumer
- [ ] Cassandra `messages` table with TWCS
- [ ] Event schema validation
- [ ] Publisher deduplication (Nats-Msg-Id)

**Success Criteria:**
- 10K messages/sec sustained write throughput
- < 5ms client response time (202 Accepted)
- Zero message loss under load

---

### Phase 2: Query Path (2-3 weeks)

**Objectives:**
- REST API compatibility with existing clients
- Efficient message history queries

**Deliverables:**
- [ ] Query Service with cursor-based pagination
- [ ] Message history from Cassandra
- [ ] Metadata from MongoDB
- [ ] Token-aware Cassandra client

**Success Criteria:**
- API compatibility with existing Rocket.Chat clients
- < 50ms P99 for message history queries
- Cursor pagination handles large channels

---

### Phase 3: Fan-Out & Notification (4-5 weeks)

**Objectives:**
- Instance-level fan-out architecture
- WebSocket connection management
- Presence tracking

**Deliverables:**
- [ ] Fan-Out Service with routing table
- [ ] Presence watcher (Redis pub/sub)
- [ ] Membership consumer
- [ ] Notification Service with WebSocket manager
- [ ] Local routing table
- [ ] Instance inbox subscription
- [ ] Typing indicator handling

**Success Criteria:**
- Support 100K+ channel memberships per user
- < 10ms event delivery latency
- Routing table cold start < 60s
- Zero per-channel NATS subscriptions

**Risk Mitigation:**
- This phase carries highest risk as core innovation
- Plan for additional debugging and optimization time
- Consider parallel workstream for fallback approach

---

### Phase 4: Workers & Webhooks (3-4 weeks)

**Objectives:**
- Complete worker pool ecosystem
- Webhook platform for integrations

**Deliverables:**
- [ ] Search Indexer worker
- [ ] Push Notification worker
- [ ] Meta Writer worker
- [ ] Webhook Dispatcher with retry logic
- [ ] Incoming webhook endpoint
- [ ] Elasticsearch cluster
- [ ] Webhook delivery log in Cassandra

**Success Criteria:**
- Search indexing keeps pace with message volume
- Push notifications delivered < 1s
- Webhook retry with exponential backoff
- HMAC signature verification working

---

### Phase 5: Migration & Cutover (3-4 weeks)

**Objectives:**
- Zero-downtime migration
- Data integrity verification
- Gradual traffic cutover

**Deliverables:**
- [ ] Historical data migration scripts
- [ ] Dual-write infrastructure
- [ ] Shadow traffic testing
- [ ] Canary deployment tooling
- [ ] Rollback procedures documented

**Success Criteria:**
- All historical messages migrated
- Dual-write consistency verified
- Canary deployment at 5% → 25% → 50% → 100%
- Rollback tested and documented

---

### Phase 6: Decommission (1-2 weeks)

**Objectives:**
- Remove legacy infrastructure
- Optimize production configuration

**Deliverables:**
- [ ] Meteor.js oplog tailing removed
- [ ] MongoDB message collections dropped
- [ ] Production performance tuning
- [ ] Documentation updated

**Success Criteria:**
- No remaining dependencies on legacy systems
- Performance meets or exceeds baseline
- Operational runbooks complete

---

### Phase 7: Multi-Region Deployment (4-6 weeks)

**Objectives:**
- Reduce latency for globally distributed users
- Deploy regional edge services
- Enable cross-region message delivery via NATS super-clusters

**Sub-phases:**

#### Phase 7.1: Regional Infrastructure (2 weeks)

**Deliverables:**
- [ ] EU-West Kubernetes cluster provisioned
- [ ] Asia-Pacific Kubernetes cluster provisioned
- [ ] NATS 3-node cluster deployed in each region
- [ ] NATS gateway links configured between regions
- [ ] Gateway link latency and health monitoring

**Success Criteria:**
- NATS gateway links operational between all regions
- Gateway link latency <100ms for US-EU, <200ms for US-Asia
- Interest-based forwarding validated

#### Phase 7.2: Data Store Replication (2 weeks)

**Deliverables:**
- [ ] Cassandra NetworkTopologyStrategy keyspace configured
- [ ] Cassandra nodes deployed in EU-West DC (3 nodes, RF=3)
- [ ] Cassandra nodes deployed in Asia DC (3 nodes, RF=3)
- [ ] Cross-DC replication validated (<200ms lag)
- [ ] Redis read replicas deployed in EU and Asia
- [ ] MongoDB secondaries deployed in EU and Asia
- [ ] Read preference tags configured for regional routing

**Success Criteria:**
- Cassandra cross-DC replication lag <200ms (p95)
- Redis replica lag <100ms
- MongoDB secondaries in sync
- Regional Query Services can read from local replicas

#### Phase 7.3: Regional Service Deployment (1-2 weeks)

**Deliverables:**
- [ ] API Gateway deployed in EU-West and Asia
- [ ] Query Service deployed in EU-West and Asia
- [ ] Notification Service deployed in EU-West (5 instances)
- [ ] Notification Service deployed in Asia (5 instances)
- [ ] GeoDNS configured for regional routing
- [ ] Health checks configured for regional failover

**Success Criteria:**
- Users routed to nearest region via GeoDNS
- Same-region message delivery <50ms (p95)
- Cross-region message delivery <300ms (p95)
- Regional query latency <30ms (p95)

#### Phase 7.4: Validation and Cutover (1 week)

**Deliverables:**
- [ ] Load testing with realistic cross-region traffic patterns
- [ ] Gateway link failover tested
- [ ] Regional NATS cluster failover tested
- [ ] Cassandra DC failover tested
- [ ] Runbooks for multi-region operations
- [ ] Monitoring dashboards for cross-region metrics

**Success Criteria:**
- All failover scenarios tested and documented
- Latency SLOs met under load
- Operations team trained on multi-region procedures

**Risk Mitigation:**
- Deploy EU-West first, validate, then add Asia
- Start with small percentage of traffic per region
- Maintain ability to route all traffic to primary region

---

### Phase 8: Regional Autonomy (4-6 weeks) — *Optional*

**Prerequisites:**
- Phase 7 fully operational and stable
- Business requirement for same-region chat during network partitions
- Team experienced with distributed systems and conflict resolution

**Objectives:**
- Enable same-region messaging during network partitions
- Implement event reconciliation after partition recovery
- Maintain eventual consistency across regions

**Sub-phases:**

#### Phase 8.1: Regional Command & Fan-Out Services (2 weeks)

**Deliverables:**
- [ ] Command Service deployed in EU-West and Asia
- [ ] Fan-Out Service deployed in EU-West and Asia
- [ ] Regional routing tables implemented (~900MB per region)
- [ ] Remote user index for cross-region routing
- [ ] Health checks detect partition and switch to autonomous mode

**Success Criteria:**
- Regional Command Service accepts writes when primary unreachable
- Regional Fan-Out Service routes to local Notification Services
- Autonomous mode activates within 10 seconds of partition detection

#### Phase 8.2: Regional NATS Streams (2 weeks)

**Deliverables:**
- [ ] Regional JetStream streams created (`MESSAGES_US`, `MESSAGES_EU`, `MESSAGES_AP`)
- [ ] Cross-region stream mirroring configured
- [ ] Region-prefixed message ID generation (`msg_{region}_{ulid}`)
- [ ] Subject hierarchy updated with region prefix
- [ ] Mirror lag monitoring and alerting

**Success Criteria:**
- Events published to regional streams
- Mirrors sync within 5 minutes of partition recovery
- No message ID collisions during partitions

#### Phase 8.3: Event Reconciliation & Conflict Resolution (1-2 weeks)

**Deliverables:**
- [ ] Fan-Out Services consume from mirror streams after reconnection
- [ ] Presence sync protocol between regional Fan-Out Services
- [ ] Last-Write-Wins conflict resolution for edits/deletes
- [ ] Client SDK handles out-of-order message arrival
- [ ] Reconciliation progress monitoring

**Success Criteria:**
- All missed events delivered within 5 minutes of reconnection
- Presence state consistent within 1 minute of reconnection
- Conflicts resolved deterministically (LWW by timestamp)

#### Phase 8.4: Validation & Chaos Testing (1 week)

**Deliverables:**
- [ ] Chaos testing: simulate 30-minute regional partition
- [ ] Verify same-region chat works during partition
- [ ] Verify reconciliation after partition recovery
- [ ] Runbooks for regional autonomy operations
- [ ] Monitoring dashboards for partition detection and recovery

**Success Criteria:**
- Same-region message delivery <50ms during partition
- All messages reconciled after 30-minute partition within 5 minutes
- No data loss or duplicate messages after reconciliation
- Operations team trained on autonomous mode procedures

**Risk Mitigation:**
- Feature flag to disable regional autonomy and fall back to global mode
- Extensive chaos testing before production enablement
- Start with single secondary region (EU), add others after validation
- Clear rollback procedure to Phase 7 architecture

---

## Rollout Strategy

### 1. Shadow Mode

Deploy new architecture alongside existing Meteor.js:
- Command Service publishes events
- Workers write to both Cassandra and MongoDB
- Fan-Out + Notification Service run in shadow mode
- Compare latency and correctness metrics

### 2. Canary Deployment

Route 5% of WebSocket connections to new Notification Service:
- Monitor delivery latency
- Track missed messages
- Validate fan-out correctness

### 3. Gradual Rollout

Increase traffic incrementally:
- 5% → 25% → 50% → 100%
- 2-week rollout period
- Continuous monitoring at each stage

### 4. Rollback Plan

Gateway routing can switch back to Meteor.js instantly:
- Dual-write ensures MongoDB remains up-to-date
- Client reconnection to legacy system is transparent
- Rollback decision within 5 minutes of detecting issues

---

## Team Allocation (Suggested)

| Phase | Team Size | Key Roles |
|-------|-----------|-----------|
| Phase 0 | 2 | DevOps, SRE |
| Phase 1-2 | 3 | Backend engineers |
| Phase 3 | 4 | Backend engineers, Systems engineer |
| Phase 4 | 3 | Backend engineers |
| Phase 5-6 | 4 | Full team, DBA support |

---

## Milestones

| Milestone | Target | Gate Criteria |
|-----------|--------|---------------|
| **M1: Infrastructure Ready** | End of Phase 0 | Clusters healthy, monitoring live |
| **M2: Write Path Validated** | End of Phase 1 | 10K msg/sec, zero loss |
| **M3: Read Path Complete** | End of Phase 2 | API compatible |
| **M4: Real-time Delivery** | End of Phase 3 | WebSocket fan-out working |
| **M5: Full Feature Parity** | End of Phase 4 | Search, push, webhooks |
| **M6: Production Cutover** | End of Phase 5 | 100% traffic on new system |
| **M7: Legacy Decommissioned** | End of Phase 6 | Clean architecture |
| **M8: Multi-Region Live** | End of Phase 7 | Global edge deployment, <50ms same-region latency |
| **M9: Regional Autonomy** *(Optional)* | End of Phase 8 | Same-region chat during partitions, event reconciliation |
