# Multi-Region Deployment

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [Regional Topology](#2-regional-topology)
3. [NATS Super-Cluster Configuration](#3-nats-super-cluster-configuration)
4. [Regional Service Deployment](#4-regional-service-deployment)
5. [Data Store Multi-DC Configuration](#5-data-store-multi-dc-configuration)
6. [Client Routing](#6-client-routing)
7. [Latency Optimization Patterns](#7-latency-optimization-patterns)
8. [Failure Scenarios and Recovery](#8-failure-scenarios-and-recovery)
9. [Deployment Checklist](#9-deployment-checklist)
10. [Future Enhancement: Regional Autonomy](#10-future-enhancement-regional-autonomy)

---

## 1. Overview

The multi-region deployment architecture enables low-latency real-time messaging for geographically distributed users. The design targets:

- **Same-region message delivery:** <50ms
- **Cross-region message delivery:** <300ms
- **History queries:** <30ms (from regional replicas)

### Design Principles

1. **Regional Edge + Global Backbone:** Users connect to geographically nearest services; global coordination happens via NATS gateway links
2. **Global Fan-Out, Regional Delivery:** The Fan-Out Service remains centralized for simplicity; Notification Services are regional
3. **Read-Local, Write-Global:** Queries read from regional replicas; writes go to primary region
4. **Graceful Degradation:** Regional failures increase latency but don't cause outages

---

## 2. Regional Topology

### Initial Deployment (3 Regions)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           NATS Super-Cluster                             │
│                                                                          │
│    US-East (Primary)         US-West              EU-West               │
│    ┌─────────────┐       ┌─────────────┐     ┌─────────────┐           │
│    │ NATS Cluster│◄─────►│ NATS Cluster│◄───►│ NATS Cluster│           │
│    │   (3 nodes) │       │   (3 nodes) │     │   (3 nodes) │           │
│    └──────┬──────┘       └──────┬──────┘     └──────┬──────┘           │
│           │                     │                   │                    │
│           │    Gateway Links (Interest-Based)       │                    │
│           │◄────────────────────┼───────────────────►                    │
│                                 │                                        │
│                          ┌──────┴──────┐                                 │
│                          │ Asia-Pacific│                                 │
│                          │ NATS Cluster│                                 │
│                          │   (3 nodes) │                                 │
│                          └─────────────┘                                 │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│   US-East (Primary) │ │      EU-West        │ │    Asia-Pacific     │
│                     │ │                     │ │                     │
│ ┌─────────────────┐ │ │ ┌─────────────────┐ │ │ ┌─────────────────┐ │
│ │  Command Svc    │ │ │ │  Query Svc      │ │ │ │  Query Svc      │ │
│ │  Fan-Out Svc    │ │ │ │  Notif Svc      │ │ │ │  Notif Svc      │ │
│ │  Query Svc      │ │ │ │  API Gateway    │ │ │ │  API Gateway    │ │
│ │  Notif Svc      │ │ │ └─────────────────┘ │ │ └─────────────────┘ │
│ │  All Workers    │ │ │                     │ │                     │
│ │  API Gateway    │ │ │ ┌─────────────────┐ │ │ ┌─────────────────┐ │
│ └─────────────────┘ │ │ │  Cassandra DC   │ │ │ │  Cassandra DC   │ │
│                     │ │ │  (replica)      │ │ │ │  (replica)      │ │
│ ┌─────────────────┐ │ │ │  RF=3           │ │ │ │  RF=3           │ │
│ │  Cassandra DC   │ │ │ └─────────────────┘ │ │ └─────────────────┘ │
│ │  (primary)      │ │ │                     │ │                     │
│ │  RF=3           │ │ │ ┌─────────────────┐ │ │ ┌─────────────────┐ │
│ └─────────────────┘ │ │ │  Redis Replica  │ │ │ │  Redis Replica  │ │
│                     │ │ └─────────────────┘ │ │ └─────────────────┘ │
│ ┌─────────────────┐ │ │                     │ │                     │
│ │  Redis Primary  │ │ │ ┌─────────────────┐ │ │ ┌─────────────────┐ │
│ │  MongoDB Primary│ │ │ │  MongoDB Sec    │ │ │ │  MongoDB Sec    │ │
│ │  Elasticsearch  │ │ │ └─────────────────┘ │ │ └─────────────────┘ │
│ └─────────────────┘ │ │                     │ │                     │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
```

### Region Roles

| Region | Role | Services | Data Stores |
|--------|------|----------|-------------|
| **US-East** | Primary | All services, all workers | Cassandra primary DC, Redis primary, MongoDB primary, Elasticsearch |
| **EU-West** | Edge | Query Service, Notification Service, API Gateway | Cassandra replica DC, Redis read replica, MongoDB secondary |
| **Asia-Pacific** | Edge | Query Service, Notification Service, API Gateway | Cassandra replica DC, Redis read replica, MongoDB secondary |
| **US-West** | Edge (optional) | Query Service, Notification Service | Cassandra replica DC, Redis read replica |

---

## 3. NATS Super-Cluster Configuration

### Gateway Configuration

Each NATS cluster connects to other regional clusters via gateway links. Gateways use **interest-based forwarding**: subscription information propagates across gateways, but messages only traverse when there's a matching subscriber.

```hcl
# nats-server.conf for US-East cluster

cluster {
  name: "us-east"
  listen: 0.0.0.0:6222
  routes: [
    "nats://nats-1.us-east:6222",
    "nats://nats-2.us-east:6222",
    "nats://nats-3.us-east:6222"
  ]
}

gateway {
  name: "us-east"
  listen: 0.0.0.0:7222

  gateways: [
    {name: "us-west", urls: ["nats://nats-gw.us-west:7222"]},
    {name: "eu-west", urls: ["nats://nats-gw.eu-west:7222"]},
    {name: "asia",    urls: ["nats://nats-gw.asia:7222"]}
  ]
}

jetstream {
  store_dir: "/data/jetstream"
  max_memory_store: 1GB
  max_file_store: 100GB
}
```

### JetStream Stream Replication

JetStream streams can be configured for cross-region replication:

```bash
# Create MESSAGES stream with cross-cluster mirroring
nats stream add MESSAGES \
  --subjects "messages.>" \
  --retention limits \
  --max-age 30d \
  --replicas 3 \
  --cluster us-east

# Create a mirror in EU for disaster recovery (optional)
nats stream add MESSAGES_EU_MIRROR \
  --mirror MESSAGES \
  --cluster eu-west
```

### Interest-Based Forwarding

When a Notification Service instance in EU subscribes to `instance.events.notif-eu-01`, the subscription propagates to the US-East gateway. When the Fan-Out Service publishes to that subject, NATS routes the message across the gateway.

**Key benefit:** Messages for channels with only US members never cross the Atlantic — there's no subscriber interest in EU.

---

## 4. Regional Service Deployment

### Notification Service (Regional)

Each region runs its own pool of Notification Service instances. Users connect to the nearest region via GeoDNS or Anycast.

```yaml
# Kubernetes deployment for EU region
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: chat-eu
spec:
  replicas: 10
  template:
    spec:
      containers:
      - name: notification-service
        env:
        - name: NATS_URL
          value: "nats://nats-cluster.eu-west:4222"
        - name: INSTANCE_REGION
          value: "eu-west"
        - name: REDIS_URL
          value: "redis://redis-replica.eu-west:6379"
```

**Instance Naming Convention:**
```
notif-{region}-{instance_number}
# Examples: notif-us-east-01, notif-eu-west-03, notif-asia-07
```

### Query Service (Regional)

Regional Query Services read from local data store replicas:

```yaml
env:
- name: CASSANDRA_CONTACT_POINTS
  value: "cassandra-1.eu-west,cassandra-2.eu-west,cassandra-3.eu-west"
- name: CASSANDRA_LOCAL_DC
  value: "eu-west"
- name: CASSANDRA_CONSISTENCY_READ
  value: "LOCAL_ONE"
- name: MONGODB_READ_PREFERENCE
  value: "secondaryPreferred"
- name: MONGODB_READ_PREFERENCE_TAGS
  value: '[{"region": "eu-west"}]'
```

### Fan-Out Service (Global — Primary Region Only)

The Fan-Out Service runs only in the primary region. It publishes to instance inboxes across all regions via NATS gateway links.

```yaml
# Only deployed in US-East
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fan-out-service
  namespace: chat-us-east
spec:
  replicas: 3
  template:
    spec:
      nodeSelector:
        topology.kubernetes.io/region: us-east-1
```

---

## 5. Data Store Multi-DC Configuration

### Apache Cassandra

**Topology:**

| DC | Nodes | Replication Factor | Purpose |
|----|-------|-------------------|---------|
| us-east | 3 | 3 | Primary (writes) |
| eu-west | 3 | 3 | Read replica |
| asia | 3 | 3 | Read replica |

**Keyspace Configuration:**

```sql
CREATE KEYSPACE chat WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'us-east': 3,
  'eu-west': 3,
  'asia': 3
};
```

**Consistency Levels:**

| Operation | Consistency Level | Rationale |
|-----------|------------------|-----------|
| Message write | LOCAL_QUORUM | Durability in primary DC |
| Message read | LOCAL_ONE | Low latency from regional replica |
| Counter update | LOCAL_QUORUM | Correctness for reply counts |

**Estimated Replication Lag:** 100-200ms between US-East and EU/Asia DCs.

### Redis/Valkey

**Topology:**
- Primary: US-East (handles all writes)
- Read Replicas: EU-West, Asia-Pacific (async replication)

```bash
# Redis replication setup on EU replica
redis-cli replicaof redis-primary.us-east 6379
```

**Regional Query Services configuration:**

```yaml
# EU Query Service
REDIS_URL: "redis://redis-replica.eu-west:6379"
REDIS_READ_ONLY: "true"

# Presence writes still go to primary via Command Service in US-East
```

### MongoDB

**Topology:** Primary-Secondary-Secondary-Secondary-Arbiter (PSSAA)

| Member | Region | Role | Priority |
|--------|--------|------|----------|
| mongo-1 | us-east | Primary | 10 |
| mongo-2 | us-east | Secondary | 9 |
| mongo-3 | eu-west | Secondary | 5 |
| mongo-4 | asia | Secondary | 5 |
| arbiter | us-east | Arbiter | 0 |

**Read Preference:**

```javascript
// Regional Query Service
db.getMongo().setReadPref("secondaryPreferred", [
  { "region": "eu-west" },  // Prefer local region
  { "region": "us-east" }   // Fallback to primary DC
]);
```

---

## 6. Client Routing

### GeoDNS Configuration

```
; DNS Zone configuration
api.chat.example.com.  300  IN  A  203.0.113.10  ; US-East (default)
api.chat.example.com.  300  IN  A  203.0.113.20  ; EU-West (geo: EU)
api.chat.example.com.  300  IN  A  203.0.113.30  ; Asia (geo: APAC)

ws.chat.example.com.   300  IN  A  203.0.113.11  ; US-East WebSocket
ws.chat.example.com.   300  IN  A  203.0.113.21  ; EU-West WebSocket
ws.chat.example.com.   300  IN  A  203.0.113.31  ; Asia WebSocket
```

### Anycast Alternative

For more precise routing, use Anycast IP addresses with regional load balancers:

```
                    User Request
                         │
                         ▼
                  ┌──────────────┐
                  │  Anycast IP  │
                  │ 198.51.100.1 │
                  └──────┬───────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │ US-East │    │ EU-West │    │  Asia   │
    │   LB    │    │   LB    │    │   LB    │
    └─────────┘    └─────────┘    └─────────┘
```

---

## 7. Latency Optimization Patterns

### Same-Region Message Flow (<50ms)

```
Alice (EU) sends message to #general (mostly EU members)

1. Alice → EU API Gateway → US-East Command Service    [~80ms]
2. Command Service → US-East NATS JetStream            [~1ms]
3. US-East NATS → Fan-Out Service                      [~1ms]
4. Fan-Out Service → NATS (instance inboxes)           [~1ms]
5. US-East NATS Gateway → EU NATS Gateway              [~80ms]
6. EU NATS → EU Notification Service                   [~1ms]
7. EU Notification Service → Bob (EU)                  [~5ms]

Total: ~170ms end-to-end

Optimization: Local event echo
- Client displays optimistic UI immediately
- Server confirms via WebSocket notification
- Perceived latency: <50ms
```

### Cross-Region Message Flow (<300ms)

```
Alice (EU) sends message to #global (US + EU + Asia members)

1. Alice → EU Gateway → US-East Command Service        [~80ms]
2. Command Service → NATS JetStream                    [~1ms]
3. Fan-Out Service routes to 3 regions:
   - US-East instances: direct (~1ms)
   - EU instances: gateway (~80ms)
   - Asia instances: gateway (~150ms)
4. Regional Notification Services → Users              [~5ms]

Total to US users: ~90ms
Total to EU users: ~170ms
Total to Asia users: ~240ms
```

### Query Latency Optimization

Regional Query Services read from local Cassandra replicas:

```
Bob (Asia) loads #general message history

1. Bob → Asia API Gateway                              [~5ms]
2. Asia API Gateway → Asia Query Service               [~1ms]
3. Asia Query Service → Asia Cassandra (LOCAL_ONE)    [~10ms]
4. Response to Bob                                     [~5ms]

Total: ~21ms (vs ~250ms if reading from US-East)
```

### Presence Update Propagation

```
Carol (Asia) comes online

1. Asia Notification Service → US-East Redis           [~150ms write]
2. US-East Redis → EU/Asia Redis replicas             [~100ms async]
3. Fan-Out Service sees presence change               [~5ms via pub/sub]
4. Fan-Out Service routes to interested instances     [~80-150ms gateway]

Total propagation: ~200-300ms for global visibility
```

---

## 8. Failure Scenarios and Recovery

### Scenario 1: Regional NATS Cluster Failure

**Impact:** Users in affected region cannot receive real-time updates.

**Detection:**
- NATS cluster health checks fail
- Gateway connection errors in other regions
- Notification Service connection failures

**Recovery:**
1. GeoDNS/LB shifts traffic to next-nearest region
2. Users reconnect to healthy region (increased latency)
3. NATS cluster recovers via RAFT leader election
4. Traffic gradually returns to recovered region

**Data Loss:** None — JetStream events buffered in primary region.

### Scenario 2: Gateway Link Failure (Network Partition)

**Impact:** Cross-region message delivery stops; same-region delivery continues.

**Detection:**
- Gateway link heartbeat failures
- Consumer lag increases for cross-region instances

**Recovery:**
1. NATS buffers messages for affected region
2. Network path restored (or rerouted via tertiary gateway)
3. Buffered messages drain automatically
4. No manual intervention required

**RTO:** Automatic recovery when network restores (typically <5 minutes).

### Scenario 3: Regional Cassandra DC Failure

**Impact:** Query Service in affected region falls back to remote DC.

**Detection:**
- Cassandra node health checks fail
- Query latency increases in affected region

**Recovery:**
1. Token-aware client routes to next-nearest DC
2. Queries served from remote DC (higher latency)
3. Failed DC nodes replaced and bootstrapped
4. Data streams back via repair

**Fallback Consistency:**
```yaml
# Dynamic consistency level adjustment
CASSANDRA_CONSISTENCY_READ: "LOCAL_ONE"
CASSANDRA_FALLBACK_CONSISTENCY: "ONE"  # Cross-DC fallback
```

### Scenario 4: Primary Region (US-East) Complete Failure

**Impact:** All writes fail; reads continue from regional replicas.

**Detection:**
- Command Service unreachable
- Fan-Out Service unreachable
- Write operations return 503

**Manual Failover Procedure:**
1. Promote EU-West to primary (MongoDB election, Cassandra DC promotion)
2. Deploy Command Service and Fan-Out Service to EU-West
3. Update NATS gateway configuration
4. DNS cutover to EU-West endpoints

**RTO:** 15-30 minutes for manual failover.

**RPO:** Potential loss of in-flight events not yet persisted to Cassandra.

### Scenario 5: Redis Primary Failure

**Impact:** Presence updates fail; stale presence data in regional replicas.

**Detection:**
- Redis primary health check failures
- Notification Service presence write errors

**Recovery:**
1. Redis Sentinel promotes replica to primary
2. Other replicas reconfigure to new primary
3. Presence data self-heals via TTL expiry (120s)

**RTO:** 10-30 seconds for Sentinel failover.

---

## 9. Deployment Checklist

### Pre-Deployment

- [ ] NATS clusters deployed in all target regions (3 nodes each)
- [ ] Gateway links configured and tested between all region pairs
- [ ] Cassandra multi-DC keyspace created with correct RF
- [ ] Redis replication configured between regions
- [ ] MongoDB replica set configured with regional members
- [ ] GeoDNS or Anycast configured for client routing
- [ ] Monitoring dashboards created for cross-region metrics

### Service Deployment

- [ ] API Gateway deployed in each region
- [ ] Notification Service deployed in each region
- [ ] Query Service deployed in each region
- [ ] Command Service deployed in primary region
- [ ] Fan-Out Service deployed in primary region
- [ ] Workers deployed in primary region

### Validation

- [ ] Same-region message delivery <50ms (p95)
- [ ] Cross-region message delivery <300ms (p95)
- [ ] Regional query latency <30ms (p95)
- [ ] Gateway link failover tested
- [ ] Regional NATS cluster failover tested
- [ ] Cassandra DC failover tested
- [ ] Load test with realistic cross-region traffic patterns

### Monitoring

- [ ] NATS gateway link latency and throughput
- [ ] Cross-region message delivery latency percentiles
- [ ] Cassandra cross-DC replication lag
- [ ] Redis replication lag
- [ ] Regional Notification Service connection counts
- [ ] Per-region error rates

---

## 10. Future Enhancement: Regional Autonomy

The current design prioritizes **consistency over availability**: all writes go through the primary region, so an isolated region cannot send messages. This section describes an optional enhancement to enable **same-region chat during network partitions**.

### 10.1 Current Limitation

When a region is isolated from the primary region:

```
Current Behavior During EU Isolation:

┌─────────────────────────────────────────────────────────────────┐
│  US-East (Primary)              ║  EU-West (Isolated)           │
│                                 ║                               │
│  ┌─────────────────┐           ║  ┌─────────────────┐          │
│  │ Command Service │ ◄─────────╬──┤ API Gateway     │ ✗ FAILS  │
│  └─────────────────┘           ║  └─────────────────┘          │
│                                 ║                               │
│  ┌─────────────────┐           ║  ┌─────────────────┐          │
│  │ Fan-Out Service │ ──────────╬─►│ Notif Service   │ ✗ NO DATA│
│  └─────────────────┘           ║  └─────────────────┘          │
└─────────────────────────────────────────────────────────────────┘

EU Users Experience:
✗ Cannot send messages (Command Service unreachable)
✗ Cannot receive new messages (Fan-Out Service unreachable)
✓ Can query message history (local Cassandra replica - stale)
✓ Can see cached presence (local Redis replica - stale)
```

### 10.2 Regional Autonomy Architecture

To enable same-region chat during isolation, deploy **Regional Command and Fan-Out Services**:

```
Regional Autonomy Architecture:

┌─────────────────────────────────────────────────────────────────┐
│  US-East (Primary)              ║  EU-West (Autonomous)         │
│                                 ║                               │
│  ┌─────────────────┐           ║  ┌─────────────────┐          │
│  │ Command Service │           ║  │ Command Service │ ✓ LOCAL  │
│  └────────┬────────┘           ║  └────────┬────────┘          │
│           ▼                     ║           ▼                   │
│  ┌─────────────────┐           ║  ┌─────────────────┐          │
│  │ Regional NATS   │◄══════════╬══►│ Regional NATS   │          │
│  │ JetStream       │  Gateway  ║  │ JetStream       │          │
│  │ MESSAGES_US     │  Links    ║  │ MESSAGES_EU     │          │
│  └────────┬────────┘           ║  └────────┬────────┘          │
│           ▼                     ║           ▼                   │
│  ┌─────────────────┐           ║  ┌─────────────────┐          │
│  │ Fan-Out Service │           ║  │ Fan-Out Service │ ✓ LOCAL  │
│  │ (US routing)    │           ║  │ (EU routing)    │          │
│  └────────┬────────┘           ║  └────────┬────────┘          │
│           ▼                     ║           ▼                   │
│  ┌─────────────────┐           ║  ┌─────────────────┐          │
│  │ Notif Services  │           ║  │ Notif Services  │          │
│  └─────────────────┘           ║  └─────────────────┘          │
└─────────────────────────────────────────────────────────────────┘

During Isolation - EU Users Can:
✓ Send messages to EU users and bots
✓ Receive messages from EU users and bots
✓ See EU user presence updates
✗ Cannot reach US/Asia users (expected)
```

### 10.3 Key Design Changes

#### Regional NATS Streams

Each region maintains its own JetStream streams for locally-originated events:

```hcl
# US-East NATS configuration
jetstream {
  store_dir: "/data/jetstream"
}

# Regional stream for US-originated messages
# nats stream add MESSAGES_US --subjects "messages.us.>" --replicas 3

# Mirror of EU stream (populated when connected)
# nats stream add MESSAGES_EU_MIRROR --mirror MESSAGES_EU --cluster eu-west
```

**Subject Hierarchy with Region Prefix:**

```
messages.{region}.send.{channel_id}      → MessageSent event
messages.{region}.edit.{channel_id}      → MessageEdited event
messages.{region}.delete.{channel_id}    → MessageDeleted event

# Examples:
messages.us.send.ch_general              → US-originated message
messages.eu.send.ch_general              → EU-originated message
```

#### Region-Prefixed Message IDs

To avoid ID collisions during partitions, message IDs include a region prefix:

```json
{
  "message_id": "msg_us_01HZ3K4M5N6P7Q8R9S0T",
  "region_origin": "us-east",
  "channel_id": "ch_global",
  "text": "Hello from US",
  "timestamp": "2026-02-01T10:30:00.123Z"
}
```

**ID Format:** `msg_{region}_{ulid}`

- `region`: 2-letter region code (us, eu, ap)
- `ulid`: Universally Unique Lexicographically Sortable Identifier

#### Regional Routing Tables

Each regional Fan-Out Service maintains a **regional routing table** for local users, plus a **remote user index** for cross-region routing:

```go
type RegionalFanOutService struct {
    region          string
    localRouting    *ChannelRouting  // Users connected to this region
    remoteIndex     *RemoteUserIndex // Users in other regions (synced via gateway)

    localStream     string           // e.g., "MESSAGES_US"
    mirrorStreams   []string         // e.g., ["MESSAGES_EU_MIRROR", "MESSAGES_AP_MIRROR"]
}

type RemoteUserIndex struct {
    mu      sync.RWMutex
    regions map[string]*RegionEntry  // region → users
}

type RegionEntry struct {
    Region    string
    Users     map[string]string      // user_id → instance_id
    LastSync  time.Time
    Connected bool                    // false during partition
}
```

**Memory Impact:**

| Structure | Size (50K users, 3 regions) |
|-----------|----------------------------|
| Local Routing Table | ~700MB (regional subset) |
| Remote User Index | ~100MB per remote region |
| **Total per Region** | ~900MB (vs 2GB global) |

### 10.4 Cross-Region Event Flow

#### Normal Operation (Connected)

```
Alice (US) sends message to #global (US + EU members):

1. Alice → US Command Service
2. US Command Service → MESSAGES_US stream (messages.us.send.ch_global)
3. US Fan-Out Service consumes event
4. US Fan-Out routes to:
   a. Local US Notification Services (direct)
   b. EU Fan-Out Service via gateway (messages.us.send.ch_global)
5. EU Fan-Out routes to EU Notification Services
6. All users receive message
```

#### During Partition (Isolated)

```
Bob (EU) sends message to #eu-team (EU-only members):

1. Bob → EU Command Service (local)
2. EU Command Service → MESSAGES_EU stream (messages.eu.send.ch_eu-team)
3. EU Fan-Out Service consumes event
4. EU Fan-Out routes to EU Notification Services
5. EU users receive message immediately

Meanwhile in US:
- US users don't see #eu-team activity (no EU members there anyway)
- Gateway link down, events buffer in NATS
```

### 10.5 Recovery and Reconciliation

When connectivity restores, the system reconciles divergent state:

#### Phase 1: Gateway Link Reconnection

```
Timeline:
T=0          T=30min        T=30:01        T=35min
│            │              │              │
▼            ▼              ▼              ▼
Partition    Reconnect      Sync           Fully
starts       detected       begins         synced

Events During Partition:
├── MESSAGES_US: msg_us_001 → msg_us_500 (500 events)
├── MESSAGES_EU: msg_eu_001 → msg_eu_200 (200 events)
└── Both regions operated independently
```

#### Phase 2: Stream Mirroring Catches Up

NATS JetStream mirrors automatically sync when gateways reconnect:

```bash
# US-East receives EU events via mirror
$ nats stream info MESSAGES_EU_MIRROR
...
Mirror Information:
  Stream: MESSAGES_EU @ eu-west
  Lag: 0           # Caught up after reconnection
  Last Seen: 1s ago
```

#### Phase 3: Fan-Out Services Process Missed Events

Each regional Fan-Out Service processes events from mirror streams:

```go
func (f *RegionalFanOutService) ProcessMirrorStream(stream string) {
    // Resume from last processed sequence
    consumer := f.nats.CreateConsumer(stream, nats.DeliverByStartSequence(f.lastSeq[stream]))

    for msg := range consumer.Messages() {
        event := parseEvent(msg)

        // Route to local users who are members of this channel
        if f.hasLocalMembers(event.ChannelID) {
            f.routeToLocalInstances(event)
        }

        f.lastSeq[stream] = msg.Sequence
        msg.Ack()
    }
}
```

#### Phase 4: Client Message Ordering

Clients receive messages ordered by **timestamp**, interleaving events from all regions:

```
Channel: #global (US + EU members)

Before Partition:
10:00:00 [US] Alice: "Starting the meeting"

During Partition (30 minutes):
├── US users see only US messages
└── EU users see only EU messages

After Reconciliation:
10:00:00 [US] Alice: "Starting the meeting"
10:05:00 [US] Carol: "I'm here"
10:05:30 [EU] Bob: "Joining from EU"        ← Appears after sync
10:10:00 [EU] Dan: "Can you hear me?"       ← Appears after sync
10:15:00 [US] Alice: "Let's wait for EU"
10:20:00 [EU] Bob: "We're back online!"     ← Appears after sync
10:25:00 [US] Carol: "Great, let's continue"
```

#### Phase 5: Presence Reconciliation

Regional Fan-Out Services exchange presence state:

```go
type PresenceSyncMessage struct {
    Region      string
    OnlineUsers map[string]InstanceInfo  // user_id → instance info
    Timestamp   time.Time
}

// On reconnection, each region broadcasts its presence state
func (f *RegionalFanOutService) BroadcastPresence() {
    sync := PresenceSyncMessage{
        Region:      f.region,
        OnlineUsers: f.localRouting.GetAllOnlineUsers(),
        Timestamp:   time.Now(),
    }
    f.nats.Publish("presence.sync."+f.region, sync)
}

// Other regions merge the presence data
func (f *RegionalFanOutService) HandlePresenceSync(sync PresenceSyncMessage) {
    f.remoteIndex.Update(sync.Region, sync.OnlineUsers, sync.Timestamp)
}
```

### 10.6 Conflict Resolution

#### Message Edit Conflicts

If the same message is edited in different regions during partition (rare edge case):

```
Scenario: Message msg_us_001 edited in both regions

US-East (10:05:00): Edit to "Hello world!"
EU-West (10:05:30): Edit to "Hello everyone!"

Resolution: Last-Write-Wins (LWW) by timestamp
Result: "Hello everyone!" (EU edit wins, later timestamp)
```

**Cassandra Implementation:**

```sql
-- Cassandra uses cell-level timestamps for LWW
UPDATE messages
SET body = 'Hello everyone!', edited_at = '2026-02-01T10:05:30Z'
WHERE channel_id = 'ch_general'
  AND bucket = '2026-02-01'
  AND message_id = msg_us_001;

-- Later timestamp wins automatically
```

#### Membership Conflicts

If a user is added/removed from a channel in different regions:

```
Scenario: User Carol and channel #project

US-East (10:05:00): Admin adds Carol to #project
EU-West (10:05:30): Different admin removes Carol from #project

Resolution: Last-Write-Wins
Result: Carol is NOT in #project (removal wins, later timestamp)
```

### 10.7 Trade-offs Summary

| Aspect | Current (Global) | Regional Autonomy |
|--------|------------------|-------------------|
| **Same-region chat during partition** | :x: No | :white_check_mark: Yes |
| **Message ordering guarantee** | :white_check_mark: Strict global | :warning: Eventually consistent |
| **Routing table size per region** | 2GB (global) | ~900MB (regional) |
| **Conflict resolution** | Not needed | Last-Write-Wins |
| **Implementation complexity** | Lower | Higher |
| **Operational complexity** | Lower | Higher |
| **Recovery time after partition** | Instant (no divergence) | Minutes (stream sync) |

### 10.8 When to Implement Regional Autonomy

**Recommended when:**
- User base is heavily distributed across regions (>30% in non-primary regions)
- Users frequently collaborate within regional teams
- Availability is prioritized over strict consistency
- Network partitions between regions are not rare

**Not recommended when:**
- Most users are in one region
- Strict message ordering is critical (e.g., financial transactions)
- Operational team is small or inexperienced with distributed systems

---

## Related Documents

- [ADR-010: Multi-Region Deployment](../adrs/ADR-010-multi-region-deployment.md) — Decision rationale
- [Detailed Design](../detailed-design.md) — Service specifications
- [Risks and Mitigations](../appendices/risks.md) — Cross-region failure risks
- [Implementation Plan](../appendices/implementation-plan.md) — Phase 7: Multi-Region, Phase 8: Regional Autonomy
