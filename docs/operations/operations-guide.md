# Operations Guide

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [Backup & Recovery](#2-backup--recovery)
3. [Disaster Recovery](#3-disaster-recovery)
4. [Database Maintenance](#4-database-maintenance)
5. [Capacity Planning](#5-capacity-planning)
6. [Auto-Scaling](#6-auto-scaling)
7. [Runbooks](#7-runbooks)
8. [Cost Optimization](#8-cost-optimization)
9. [SLA & SLO](#9-sla--slo)

---

## 1. Overview

This document defines operational procedures for the chat platform, including backup strategies, disaster recovery, maintenance windows, and capacity planning.

### Operational Principles

| Principle | Description |
|-----------|-------------|
| **Automate Everything** | Manual operations are error-prone; automate all routine tasks |
| **Immutable Infrastructure** | Replace, don't repair; use infrastructure-as-code |
| **Observable Systems** | If you can't measure it, you can't manage it |
| **Graceful Degradation** | Partial availability is better than total outage |
| **Blast Radius Containment** | Failures should not cascade across components |

### Service Criticality Tiers

| Tier | Services | RTO | RPO | Impact |
|------|----------|-----|-----|--------|
| **Tier 1** | API Gateway, Notification Service | 5 min | 0 | Complete outage |
| **Tier 2** | Command Service, Fan-Out Service | 15 min | 0 | No new messages |
| **Tier 3** | Query Service, Workers | 30 min | 5 min | Degraded read, delayed processing |
| **Tier 4** | Search, Analytics | 2 hr | 1 hr | Feature unavailable |

---

## 2. Backup & Recovery

### 2.1 Backup Strategy Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Backup Architecture                                │
│                                                                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                   │
│  │  Cassandra  │     │   MongoDB   │     │    NATS     │                   │
│  │             │     │             │     │  JetStream  │                   │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘                   │
│         │                   │                   │                          │
│         ▼                   ▼                   ▼                          │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                   │
│  │  Snapshot   │     │  mongodump  │     │   Stream    │                   │
│  │  + Commit   │     │  + Oplog    │     │  Snapshot   │                   │
│  │    Logs     │     │             │     │             │                   │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘                   │
│         │                   │                   │                          │
│         └───────────────────┼───────────────────┘                          │
│                             │                                               │
│                             ▼                                               │
│                    ┌─────────────────┐                                      │
│                    │   S3 Backup     │                                      │
│                    │   Bucket        │                                      │
│                    │                 │                                      │
│                    │ • Versioning    │                                      │
│                    │ • Cross-region  │                                      │
│                    │ • Lifecycle     │                                      │
│                    └─────────────────┘                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Cassandra Backup

**Strategy:** Incremental snapshots + commit log archiving

| Backup Type | Frequency | Retention | Method |
|-------------|-----------|-----------|--------|
| Full Snapshot | Daily (2 AM UTC) | 30 days | `nodetool snapshot` |
| Incremental | Hourly | 7 days | Commit log archiving |
| Point-in-Time | Continuous | 24 hours | Commit logs |

**Backup Script:**
```bash
#!/bin/bash
# /opt/scripts/cassandra-backup.sh

SNAPSHOT_NAME="daily-$(date +%Y%m%d-%H%M%S)"
BACKUP_BUCKET="s3://chat-backups/cassandra"
KEYSPACE="chat"

# Create snapshot
nodetool snapshot -t "$SNAPSHOT_NAME" "$KEYSPACE"

# Find and upload snapshot files
SNAPSHOT_DIR="/var/lib/cassandra/data/$KEYSPACE"
for table_dir in "$SNAPSHOT_DIR"/*/snapshots/"$SNAPSHOT_NAME"; do
    TABLE=$(basename $(dirname $(dirname "$table_dir")))
    aws s3 sync "$table_dir" "$BACKUP_BUCKET/$SNAPSHOT_NAME/$TABLE/" \
        --storage-class STANDARD_IA
done

# Upload schema
cqlsh -e "DESCRIBE KEYSPACE $KEYSPACE" > /tmp/schema.cql
aws s3 cp /tmp/schema.cql "$BACKUP_BUCKET/$SNAPSHOT_NAME/schema.cql"

# Clear old local snapshots (keep last 3)
nodetool clearsnapshot -t "$SNAPSHOT_NAME" "$KEYSPACE"

# Log completion
echo "$(date): Backup $SNAPSHOT_NAME completed" >> /var/log/cassandra-backup.log
```

**Commit Log Archiving:**
```yaml
# cassandra.yaml
commitlog_archiving:
  archive_command: '/opt/scripts/archive-commitlog.sh %path %name'
  restore_command: '/opt/scripts/restore-commitlog.sh %from %to'
  restore_directories: /var/lib/cassandra/commitlog_restore
  restore_point_in_time: 2026-02-01T10:30:00Z
```

**Recovery Procedure:**
```bash
#!/bin/bash
# Restore Cassandra from backup

SNAPSHOT_NAME="$1"
TARGET_NODE="$2"

# 1. Stop Cassandra
systemctl stop cassandra

# 2. Clear existing data
rm -rf /var/lib/cassandra/data/chat/*

# 3. Download snapshot
aws s3 sync "s3://chat-backups/cassandra/$SNAPSHOT_NAME" /tmp/restore/

# 4. Restore schema
cqlsh -f /tmp/restore/schema.cql

# 5. Copy data files
for table in /tmp/restore/*/; do
    TABLE_NAME=$(basename "$table")
    TABLE_DIR=$(ls -d /var/lib/cassandra/data/chat/"$TABLE_NAME"-*)
    cp -r "$table"/* "$TABLE_DIR"/
done

# 6. Fix ownership
chown -R cassandra:cassandra /var/lib/cassandra/data/

# 7. Start Cassandra
systemctl start cassandra

# 8. Run repair
nodetool repair chat
```

### 2.3 MongoDB Backup

**Strategy:** Continuous oplog backup + periodic full dump

| Backup Type | Frequency | Retention | Method |
|-------------|-----------|-----------|--------|
| Full Dump | Daily (3 AM UTC) | 30 days | `mongodump` |
| Oplog Backup | Continuous | 72 hours | Oplog tailing |
| Point-in-Time | Continuous | 72 hours | Oplog replay |

**Backup Script:**
```bash
#!/bin/bash
# /opt/scripts/mongodb-backup.sh

BACKUP_DIR="/tmp/mongodb-backup-$(date +%Y%m%d)"
BACKUP_BUCKET="s3://chat-backups/mongodb"

# Full dump with oplog
mongodump \
    --uri="mongodb://backup-user:$MONGO_PASSWORD@mongodb-primary:27017" \
    --oplog \
    --gzip \
    --out="$BACKUP_DIR"

# Upload to S3
aws s3 sync "$BACKUP_DIR" "$BACKUP_BUCKET/$(date +%Y%m%d)/" \
    --storage-class STANDARD_IA

# Cleanup
rm -rf "$BACKUP_DIR"
```

**Point-in-Time Recovery:**
```bash
#!/bin/bash
# Restore MongoDB to specific point in time

BACKUP_DATE="$1"
RESTORE_TIME="$2"  # Format: 1706789400 (Unix timestamp)

# 1. Download backup
aws s3 sync "s3://chat-backups/mongodb/$BACKUP_DATE" /tmp/restore/

# 2. Restore with oplog replay to specific time
mongorestore \
    --uri="mongodb://admin:$MONGO_PASSWORD@mongodb-primary:27017" \
    --oplogReplay \
    --oplogLimit="$RESTORE_TIME:1" \
    --gzip \
    /tmp/restore/
```

### 2.4 NATS JetStream Backup

**Strategy:** Stream snapshots + consumer state backup

| Backup Type | Frequency | Retention | Method |
|-------------|-----------|-----------|--------|
| Stream Snapshot | Every 6 hours | 7 days | `nats stream backup` |
| Consumer State | Hourly | 24 hours | State export |

**Backup Script:**
```bash
#!/bin/bash
# /opt/scripts/nats-backup.sh

STREAMS=("MESSAGES" "META" "NOTIFICATIONS" "WEBHOOKS" "FILES")
BACKUP_BUCKET="s3://chat-backups/nats"
BACKUP_DIR="/tmp/nats-backup-$(date +%Y%m%d-%H%M%S)"

mkdir -p "$BACKUP_DIR"

for STREAM in "${STREAMS[@]}"; do
    # Backup stream
    nats stream backup "$STREAM" "$BACKUP_DIR/$STREAM" \
        --server nats://nats-1:4222
done

# Backup consumer state
nats consumer ls --all --json > "$BACKUP_DIR/consumers.json"

# Upload to S3
tar -czf "$BACKUP_DIR.tar.gz" "$BACKUP_DIR"
aws s3 cp "$BACKUP_DIR.tar.gz" "$BACKUP_BUCKET/$(date +%Y%m%d-%H%M%S).tar.gz"

# Cleanup
rm -rf "$BACKUP_DIR" "$BACKUP_DIR.tar.gz"
```

### 2.5 Redis Backup

**Strategy:** RDB snapshots + AOF persistence

| Backup Type | Frequency | Retention | Method |
|-------------|-----------|-----------|--------|
| RDB Snapshot | Every 15 min | 24 hours | `BGSAVE` |
| AOF | Continuous | N/A | Append-only file |
| S3 Upload | Hourly | 7 days | Copy RDB to S3 |

**Configuration:**
```conf
# redis.conf
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec

dir /var/lib/redis
dbfilename dump.rdb
appendfilename "appendonly.aof"
```

### 2.6 S3 File Backup

**Strategy:** Cross-region replication + versioning

```hcl
# terraform - S3 bucket configuration
resource "aws_s3_bucket" "uploads" {
  bucket = "chat-uploads"
}

resource "aws_s3_bucket_versioning" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_replication_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "cross-region-replication"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.uploads_replica.arn
      storage_class = "STANDARD_IA"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    id     = "archive-old-versions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}
```

### 2.7 Backup Verification

**Daily Verification Job:**
```yaml
# kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-verification
spec:
  schedule: "0 6 * * *"  # 6 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: verify
            image: chat/backup-tools:latest
            command:
            - /opt/scripts/verify-backups.sh
            env:
            - name: SLACK_WEBHOOK
              valueFrom:
                secretKeyRef:
                  name: alerts
                  key: slack-webhook
```

**Verification Script:**
```bash
#!/bin/bash
# verify-backups.sh

ERRORS=()

# Check Cassandra backup
LATEST_CASSANDRA=$(aws s3 ls s3://chat-backups/cassandra/ | tail -1 | awk '{print $2}')
if [[ -z "$LATEST_CASSANDRA" ]]; then
    ERRORS+=("Cassandra: No backup found")
else
    AGE=$(( ($(date +%s) - $(date -d "${LATEST_CASSANDRA:0:8}" +%s)) / 3600 ))
    if [[ $AGE -gt 25 ]]; then
        ERRORS+=("Cassandra: Backup is $AGE hours old")
    fi
fi

# Check MongoDB backup
LATEST_MONGO=$(aws s3 ls s3://chat-backups/mongodb/ | tail -1 | awk '{print $2}')
if [[ -z "$LATEST_MONGO" ]]; then
    ERRORS+=("MongoDB: No backup found")
fi

# Check NATS backup
LATEST_NATS=$(aws s3 ls s3://chat-backups/nats/ | tail -1 | awk '{print $4}')
if [[ -z "$LATEST_NATS" ]]; then
    ERRORS+=("NATS: No backup found")
fi

# Report results
if [[ ${#ERRORS[@]} -gt 0 ]]; then
    curl -X POST "$SLACK_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "{\"text\": \"🚨 Backup Verification Failed:\n${ERRORS[*]}\"}"
    exit 1
else
    echo "All backups verified successfully"
fi
```

---

## 3. Disaster Recovery

### 3.1 DR Strategy

| Scenario | Strategy | RTO | RPO |
|----------|----------|-----|-----|
| **Single Node Failure** | Automatic failover | < 1 min | 0 |
| **Availability Zone Failure** | Cross-AZ failover | < 5 min | 0 |
| **Region Failure** | Cross-region failover | < 30 min | < 5 min |
| **Data Corruption** | Point-in-time recovery | < 2 hr | < 1 hr |
| **Complete Data Loss** | Full restore from backup | < 4 hr | < 24 hr |

### 3.2 Multi-AZ Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              US-East Region                                  │
│                                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │      AZ-1           │  │       AZ-2          │  │       AZ-3          │ │
│  │                     │  │                     │  │                     │ │
│  │  ┌─────────────┐    │  │  ┌─────────────┐    │  │  ┌─────────────┐    │ │
│  │  │ Cassandra-1 │    │  │  │ Cassandra-2 │    │  │  │ Cassandra-3 │    │ │
│  │  └─────────────┘    │  │  └─────────────┘    │  │  └─────────────┘    │ │
│  │                     │  │                     │  │                     │ │
│  │  ┌─────────────┐    │  │  ┌─────────────┐    │  │  ┌─────────────┐    │ │
│  │  │  MongoDB-P  │    │  │  │  MongoDB-S  │    │  │  │  MongoDB-S  │    │ │
│  │  └─────────────┘    │  │  └─────────────┘    │  │  └─────────────┘    │ │
│  │                     │  │                     │  │                     │ │
│  │  ┌─────────────┐    │  │  ┌─────────────┐    │  │  ┌─────────────┐    │ │
│  │  │   NATS-1    │    │  │  │   NATS-2    │    │  │  │   NATS-3    │    │ │
│  │  └─────────────┘    │  │  └─────────────┘    │  │  └─────────────┘    │ │
│  │                     │  │                     │  │                     │ │
│  │  ┌─────────────┐    │  │  ┌─────────────┐    │  │  ┌─────────────┐    │ │
│  │  │  Redis-P    │    │  │  │  Redis-R    │    │  │  │  Redis-R    │    │ │
│  │  └─────────────┘    │  │  └─────────────┘    │  │  └─────────────┘    │ │
│  │                     │  │                     │  │                     │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Region Failover Procedure

**Pre-requisites:**
- Secondary region (EU-West) fully provisioned
- Data replication configured and healthy
- DNS TTL set to 60 seconds

**Failover Steps:**

```bash
#!/bin/bash
# /opt/scripts/region-failover.sh

PRIMARY_REGION="us-east-1"
SECONDARY_REGION="eu-west-1"

echo "=== REGION FAILOVER INITIATED ==="
echo "Failing over from $PRIMARY_REGION to $SECONDARY_REGION"

# 1. Verify secondary region health
echo "Step 1: Verifying secondary region health..."
kubectl --context="$SECONDARY_REGION" get pods -n chat | grep -v Running && {
    echo "ERROR: Secondary region not healthy"
    exit 1
}

# 2. Stop accepting new connections in primary
echo "Step 2: Draining primary region..."
kubectl --context="$PRIMARY_REGION" scale deployment api-gateway --replicas=0 -n chat

# 3. Wait for in-flight requests
echo "Step 3: Waiting for in-flight requests (30s)..."
sleep 30

# 4. Verify replication caught up
echo "Step 4: Verifying replication lag..."
CASSANDRA_LAG=$(check_cassandra_replication_lag)
MONGO_LAG=$(check_mongo_replication_lag)

if [[ $CASSANDRA_LAG -gt 60 || $MONGO_LAG -gt 60 ]]; then
    echo "WARNING: Replication lag detected. Cassandra: ${CASSANDRA_LAG}s, MongoDB: ${MONGO_LAG}s"
    read -p "Continue with potential data loss? (y/n) " -n 1 -r
    [[ ! $REPLY =~ ^[Yy]$ ]] && exit 1
fi

# 5. Promote secondary databases
echo "Step 5: Promoting secondary databases..."

# MongoDB: Force reconfiguration
kubectl --context="$SECONDARY_REGION" exec -n chat mongodb-0 -- \
    mongosh --eval "rs.reconfig({...rs.conf(), members: [{_id: 0, host: 'mongodb-0.eu-west:27017', priority: 2}]}, {force: true})"

# 6. Update DNS
echo "Step 6: Updating DNS..."
aws route53 change-resource-record-sets \
    --hosted-zone-id "$HOSTED_ZONE_ID" \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "api.chat.example.com",
                "Type": "A",
                "AliasTarget": {
                    "HostedZoneId": "'$EU_ALB_ZONE_ID'",
                    "DNSName": "'$EU_ALB_DNS'",
                    "EvaluateTargetHealth": true
                }
            }
        }]
    }'

# 7. Verify failover
echo "Step 7: Verifying failover..."
sleep 60
curl -s https://api.chat.example.com/health | grep -q "ok" && {
    echo "=== FAILOVER COMPLETE ==="
} || {
    echo "ERROR: Health check failed post-failover"
    exit 1
}
```

### 3.4 Data Corruption Recovery

**Scenario:** Messages table corrupted due to bug or attack

```bash
#!/bin/bash
# Point-in-time recovery for Cassandra

CORRUPTION_TIME="2026-02-01T10:30:00Z"
SAFE_TIME=$(date -d "$CORRUPTION_TIME - 5 minutes" +%Y-%m-%dT%H:%M:%SZ)

echo "Recovering to $SAFE_TIME (5 minutes before corruption)"

# 1. Stop workers writing to messages table
kubectl scale deployment message-writer --replicas=0 -n chat

# 2. Find backup just before corruption
BACKUP_DATE=$(aws s3 ls s3://chat-backups/cassandra/ | \
    awk '{print $2}' | \
    while read d; do
        [[ "$d" < "${SAFE_TIME:0:8}" ]] && echo "$d"
    done | tail -1)

# 3. Restore from backup
./restore-cassandra.sh "$BACKUP_DATE"

# 4. Replay commit logs up to safe time
nodetool flush
cassandra-restore-commitlog --restore-point-in-time="$SAFE_TIME"

# 5. Verify data integrity
cqlsh -e "SELECT COUNT(*) FROM chat.messages WHERE bucket='2026-02-01';"

# 6. Resume operations
kubectl scale deployment message-writer --replicas=3 -n chat
```

### 3.5 DR Testing Schedule

| Test Type | Frequency | Duration | Scope |
|-----------|-----------|----------|-------|
| Backup Restore Test | Monthly | 4 hours | Single table |
| AZ Failover Drill | Quarterly | 2 hours | Full stack |
| Region Failover Drill | Bi-annually | 4 hours | Full stack |
| Chaos Engineering | Weekly | 1 hour | Random component |

---

## 4. Database Maintenance

### 4.1 Cassandra Maintenance

**Daily Tasks:**
| Task | Schedule | Command |
|------|----------|---------|
| Repair | 2 AM (rolling) | `nodetool repair -pr` |
| Cleanup | 3 AM | `nodetool cleanup` |
| Compaction Check | Every hour | Monitor pending compactions |

**Weekly Tasks:**
| Task | Schedule | Command |
|------|----------|---------|
| Full Repair | Sunday 2 AM | `nodetool repair` |
| Tombstone Cleanup | Saturday 3 AM | `nodetool garbagecollect` |

**Compaction Monitoring:**
```yaml
# Prometheus alert
groups:
  - name: cassandra
    rules:
      - alert: CassandraHighPendingCompactions
        expr: cassandra_compaction_pending_tasks > 50
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High pending compactions on {{ $labels.instance }}"

      - alert: CassandraCompactionStuck
        expr: increase(cassandra_compaction_completed_tasks_total[1h]) == 0
              and cassandra_compaction_pending_tasks > 0
        for: 30m
        labels:
          severity: critical
```

**Repair Script:**
```bash
#!/bin/bash
# /opt/scripts/cassandra-repair.sh
# Run as rolling repair across cluster

NODES=$(nodetool status | grep "UN" | awk '{print $2}')
KEYSPACE="chat"

for NODE in $NODES; do
    echo "$(date): Starting repair on $NODE"

    # Repair only primary ranges (-pr) to avoid redundant work
    nodetool -h "$NODE" repair -pr "$KEYSPACE"

    # Wait for repair to complete
    while nodetool -h "$NODE" compactionstats | grep -q "Repair"; do
        sleep 60
    done

    echo "$(date): Repair completed on $NODE"

    # Wait before next node to reduce cluster load
    sleep 300
done
```

### 4.2 MongoDB Maintenance

**Index Maintenance:**
```javascript
// Monthly index rebuild for large collections
db.adminCommand({
    setParameter: 1,
    maxIndexBuildMemoryUsageMegabytes: 500
});

// Rebuild indexes on messages collection
db.messages.reIndex();

// Analyze index usage
db.messages.aggregate([
    { $indexStats: {} }
]).forEach(function(idx) {
    if (idx.accesses.ops < 100) {
        print("Unused index: " + idx.name);
    }
});
```

**Compact Collections:**
```bash
# Monthly compaction for collections with high churn
mongo --eval '
    ["users", "channels", "bots"].forEach(function(coll) {
        db.runCommand({ compact: coll });
    });
'
```

### 4.3 Redis Maintenance

**Memory Management:**
```bash
#!/bin/bash
# /opt/scripts/redis-maintenance.sh

# Check memory fragmentation
FRAG=$(redis-cli INFO memory | grep "mem_fragmentation_ratio" | cut -d: -f2)
if (( $(echo "$FRAG > 1.5" | bc -l) )); then
    echo "High fragmentation: $FRAG, triggering MEMORY DOCTOR"
    redis-cli MEMORY DOCTOR
fi

# Check for big keys
redis-cli --bigkeys --memkeys 2>/dev/null | tail -20

# Clean expired keys proactively
redis-cli DEBUG SLEEP 0  # Trigger lazy expiration
```

### 4.4 NATS JetStream Maintenance

**Stream Health Check:**
```bash
#!/bin/bash
# /opt/scripts/nats-health.sh

STREAMS=("MESSAGES" "META" "NOTIFICATIONS" "WEBHOOKS")

for STREAM in "${STREAMS[@]}"; do
    INFO=$(nats stream info "$STREAM" --json)

    # Check consumer lag
    LAG=$(echo "$INFO" | jq '.state.consumer_count')
    PENDING=$(echo "$INFO" | jq '.state.messages')

    echo "Stream $STREAM: $PENDING pending messages, $LAG consumers"

    # Alert if lag > 10000 messages
    if [[ $PENDING -gt 10000 ]]; then
        echo "WARNING: High message backlog in $STREAM"
    fi
done
```

### 4.5 Maintenance Windows

| Day | Time (UTC) | Maintenance Type |
|-----|------------|------------------|
| Sunday | 02:00-04:00 | Full Cassandra repair |
| Tuesday | 03:00-04:00 | MongoDB index maintenance |
| Thursday | 03:00-04:00 | Redis memory optimization |
| Saturday | 02:00-03:00 | NATS stream purge (old messages) |

---

## 5. Capacity Planning

### 5.1 Current Capacity Model

| Resource | Current | Max Capacity | Threshold |
|----------|---------|--------------|-----------|
| **Cassandra Disk** | 500 GB | 2 TB | 70% |
| **Cassandra CPU** | 4 cores | 16 cores | 75% |
| **MongoDB Disk** | 100 GB | 500 GB | 70% |
| **Redis Memory** | 8 GB | 32 GB | 80% |
| **NATS Disk** | 50 GB | 200 GB | 70% |

### 5.2 Growth Projections

```
Messages per day: 5M → 10M (6 months) → 25M (12 months)
Storage growth: ~50 GB/month
Users: 50K → 100K (6 months) → 250K (12 months)
```

### 5.3 Capacity Alerts

```yaml
# Prometheus alerts for capacity
groups:
  - name: capacity
    rules:
      - alert: CassandraDiskNearFull
        expr: (cassandra_disk_used_bytes / cassandra_disk_total_bytes) > 0.7
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Cassandra disk usage above 70%"

      - alert: RedisMemoryNearFull
        expr: (redis_memory_used_bytes / redis_memory_max_bytes) > 0.8
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Redis memory usage above 80%"

      - alert: MessageGrowthAnomaly
        expr: |
          rate(cassandra_messages_count[1h]) >
          1.5 * avg_over_time(rate(cassandra_messages_count[1h])[7d:1h])
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Message volume 50% above 7-day average"
```

---

## 6. Auto-Scaling

### 6.1 Kubernetes HPA Configuration

```yaml
# Notification Service HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: notification-service
  namespace: chat
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: notification-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: websocket_connections
      target:
        type: AverageValue
        averageValue: "8000"  # Scale when avg > 8K connections
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 5
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 120
```

```yaml
# Message Writer HPA (scale on consumer lag)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: message-writer
  namespace: chat
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: message-writer
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: nats_consumer_pending_messages
        selector:
          matchLabels:
            consumer: msg-writer-pool
      target:
        type: AverageValue
        averageValue: "1000"  # Scale when backlog > 1K per pod
```

### 6.2 Database Auto-Scaling

**Cassandra (Manual with Automation):**
```bash
#!/bin/bash
# Add node when disk > 60% across cluster

AVG_DISK=$(kubectl exec -n chat cassandra-0 -- nodetool status | \
    grep "UN" | awk '{sum+=$6; count++} END {print sum/count}')

if (( $(echo "$AVG_DISK > 60" | bc -l) )); then
    echo "Triggering Cassandra scale-up"

    # Add new node via StatefulSet
    kubectl scale statefulset cassandra --replicas=$((CURRENT_REPLICAS + 1)) -n chat

    # Wait for node to join
    sleep 300

    # Run cleanup on existing nodes to redistribute data
    for pod in $(kubectl get pods -n chat -l app=cassandra -o name); do
        kubectl exec -n chat "$pod" -- nodetool cleanup
    done
fi
```

---

## 7. Runbooks

### 7.1 Runbook Index

| Runbook | Trigger | Severity |
|---------|---------|----------|
| [RB-001: API Gateway 5xx Spike](#rb-001) | Error rate > 1% | Critical |
| [RB-002: Cassandra Node Down](#rb-002) | Node unreachable | High |
| [RB-003: NATS Consumer Lag](#rb-003) | Lag > 10K messages | Medium |
| [RB-004: WebSocket Connection Spike](#rb-004) | Connections > 50K/instance | High |
| [RB-005: Redis Memory Full](#rb-005) | Memory > 90% | Critical |

### 7.2 RB-001: API Gateway 5xx Spike {#rb-001}

**Symptoms:**
- 5xx error rate > 1%
- Increased latency
- User complaints

**Diagnosis:**
```bash
# 1. Check which backend is failing
kubectl logs -n chat -l app=api-gateway --tail=100 | grep "502\|503\|504"

# 2. Check backend health
kubectl get pods -n chat | grep -v Running

# 3. Check resource usage
kubectl top pods -n chat
```

**Resolution:**
1. If specific backend down → Restart pods
2. If resource exhaustion → Scale up
3. If dependency failure → Check databases

### 7.3 RB-002: Cassandra Node Down {#rb-002}

**Symptoms:**
- Node shows "DN" in `nodetool status`
- Increased latency on queries
- Alerts firing

**Diagnosis:**
```bash
# 1. Check node status
nodetool status

# 2. Check system logs
kubectl logs cassandra-X -n chat --tail=200

# 3. Check disk space
kubectl exec cassandra-X -n chat -- df -h
```

**Resolution:**
1. If disk full → Expand PVC or cleanup
2. If OOM → Increase memory limits
3. If hardware failure → Replace node

---

## 8. Cost Optimization

### 8.1 Resource Right-Sizing

| Service | Current | Recommended | Savings |
|---------|---------|-------------|---------|
| Command Service | 4 CPU, 8 GB | 2 CPU, 4 GB | 50% |
| Query Service | 4 CPU, 8 GB | 2 CPU, 4 GB | 50% |
| Message Writer | 2 CPU, 4 GB | 1 CPU, 2 GB | 50% |

### 8.2 Storage Tiering

| Data Age | Storage Class | Cost |
|----------|---------------|------|
| < 30 days | SSD (gp3) | $0.08/GB |
| 30-90 days | HDD (st1) | $0.045/GB |
| > 90 days | S3 Glacier | $0.004/GB |

### 8.3 Reserved Capacity

| Resource | On-Demand | Reserved (1yr) | Savings |
|----------|-----------|----------------|---------|
| Cassandra nodes (3x r6i.xlarge) | $7,884/yr | $4,730/yr | 40% |
| Redis (r6g.large) | $2,102/yr | $1,261/yr | 40% |

---

## 9. SLA & SLO

### 9.1 Service Level Objectives

| Metric | SLO | Measurement |
|--------|-----|-------------|
| **Availability** | 99.9% | Successful requests / Total requests |
| **Message Delivery Latency** | p99 < 500ms | Time from send to recipient delivery |
| **API Response Time** | p95 < 200ms | Time to first byte |
| **Message Durability** | 99.999999% | Messages not lost |

### 9.2 Error Budget

```
Monthly error budget = 100% - 99.9% = 0.1%
Monthly minutes = 43,200
Allowed downtime = 43.2 minutes/month
```

### 9.3 SLO Dashboards

```yaml
# Grafana dashboard query for availability
- expr: |
    sum(rate(http_requests_total{status!~"5.."}[5m])) /
    sum(rate(http_requests_total[5m])) * 100
  legend: "Availability %"

# Error budget remaining
- expr: |
    (1 - (
      sum(increase(http_requests_total{status=~"5.."}[30d])) /
      sum(increase(http_requests_total[30d]))
    )) / 0.001 * 100
  legend: "Error Budget Remaining %"
```

---

## Related Documents

- [Detailed Design](../detailed-design.md) — Service architecture
- [Security Architecture](../security/security-architecture.md) — Security controls
- [Multi-Region](../features/multi-region.md) — Multi-region deployment
- [Risks](../appendices/risks.md) — Risk assessment
