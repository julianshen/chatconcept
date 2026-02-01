# Fan-Out Service Performance Testing Strategy

**Author:** Architecture Team
**Status:** Draft
**Last Updated:** 2026-02-01

---

## Table of Contents

1. [Overview](#1-overview)
2. [Performance Objectives](#2-performance-objectives)
3. [Key Metrics](#3-key-metrics)
4. [Test Environment](#4-test-environment)
5. [Test Scenarios](#5-test-scenarios)
6. [Test Implementation](#6-test-implementation)
7. [Monitoring & Observability](#7-monitoring--observability)
8. [Success Criteria](#8-success-criteria)
9. [Bottleneck Analysis](#9-bottleneck-analysis)
10. [Performance Runbook](#10-performance-runbook)

---

## 1. Overview

The Fan-Out Service is the critical path component that routes channel events to Notification Service instances. It maintains an in-memory routing table mapping channels to instances and must handle:

- **50,000+ messages/second** throughput
- **100,000 concurrent online users**
- **~2GB routing table** in memory
- **Sub-millisecond routing lookups**
- **Thousands of presence changes/second**

### Why Performance Testing Matters

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    Fan-Out Service Critical Path                            │
│                                                                             │
│  NATS JetStream ─────► Fan-Out Service ─────► Notification Services        │
│  (50K msg/sec)         (routing table)        (100+ instances)             │
│                              │                                              │
│                              ▼                                              │
│                     If Fan-Out is slow:                                     │
│                     • Message delivery latency increases                    │
│                     • JetStream consumer lag grows                          │
│                     • Backpressure affects Command Service                  │
│                     • User experience degrades                              │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Performance Objectives

### 2.1 Service Level Objectives (SLOs)

| Metric | Target | Critical Threshold |
|--------|--------|-------------------|
| Message routing latency (p50) | < 1ms | < 5ms |
| Message routing latency (p99) | < 5ms | < 20ms |
| Message routing latency (p99.9) | < 20ms | < 100ms |
| Throughput (sustained) | 50,000 msg/sec | 40,000 msg/sec |
| Throughput (burst) | 100,000 msg/sec | 75,000 msg/sec |
| JetStream consumer lag | < 1,000 msgs | < 10,000 msgs |
| Presence update processing | < 50ms | < 200ms |
| Memory usage (steady state) | < 3GB | < 4GB |
| GC pause time (p99) | < 10ms | < 50ms |

### 2.2 Scale Parameters

| Parameter | Baseline | Target | Stress |
|-----------|----------|--------|--------|
| Online users | 10,000 | 100,000 | 500,000 |
| Active channels | 50,000 | 500,000 | 2,000,000 |
| Channels per user (avg) | 100 | 1,000 | 10,000 |
| Notification Service instances | 10 | 100 | 500 |
| Message rate | 5,000/sec | 50,000/sec | 200,000/sec |
| Presence changes | 100/sec | 1,000/sec | 10,000/sec |

---

## 3. Key Metrics

### 3.1 Throughput Metrics

```yaml
# Prometheus metrics exposed by Fan-Out Service
fanout_messages_processed_total:
  type: counter
  labels: [stream, result]
  description: Total messages processed from JetStream

fanout_messages_routed_total:
  type: counter
  labels: [target_instance]
  description: Total messages routed to each instance

fanout_batch_size:
  type: histogram
  buckets: [1, 10, 50, 100, 200, 500]
  description: JetStream fetch batch sizes

fanout_publishes_per_message:
  type: histogram
  buckets: [1, 2, 5, 10, 20, 50, 100]
  description: Instance publishes per source message
```

### 3.2 Latency Metrics

```yaml
fanout_routing_latency_seconds:
  type: histogram
  buckets: [0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1]
  labels: [operation]  # lookup, publish, total
  description: Time to route a single message

fanout_batch_processing_latency_seconds:
  type: histogram
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
  description: Time to process entire batch

fanout_presence_update_latency_seconds:
  type: histogram
  buckets: [0.001, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
  description: Time to update routing table for presence change
```

### 3.3 Resource Metrics

```yaml
fanout_routing_table_channels:
  type: gauge
  description: Number of channels in routing table

fanout_routing_table_users:
  type: gauge
  description: Number of users in routing table

fanout_routing_table_bytes:
  type: gauge
  description: Estimated memory usage of routing table

fanout_jetstream_consumer_lag:
  type: gauge
  labels: [stream, consumer]
  description: Number of pending messages in consumer

fanout_rwmutex_wait_seconds:
  type: histogram
  buckets: [0.00001, 0.0001, 0.001, 0.01, 0.1]
  labels: [lock_type]  # read, write
  description: Time waiting to acquire routing table lock
```

---

## 4. Test Environment

### 4.1 Infrastructure Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Performance Test Environment                             │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        Load Generation Cluster                        │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │ k6 Runner 1 │  │ k6 Runner 2 │  │ k6 Runner 3 │  │ k6 Runner N │  │   │
│  │  │ (messages)  │  │ (messages)  │  │ (presence)  │  │ (membership)│  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │   │
│  └─────────┼────────────────┼────────────────┼────────────────┼─────────┘   │
│            │                │                │                │             │
│            ▼                ▼                ▼                ▼             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      NATS JetStream Cluster                           │   │
│  │                         (3 nodes, R=3)                                │   │
│  └───────────────────────────────┬──────────────────────────────────────┘   │
│                                  │                                          │
│                                  ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                  Fan-Out Service Under Test                           │   │
│  │                                                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │  │  Routing Table (pre-populated)                                  │ │   │
│  │  │  • 500K channels                                                │ │   │
│  │  │  • 100K users                                                   │ │   │
│  │  │  • ~2GB memory                                                  │ │   │
│  │  └─────────────────────────────────────────────────────────────────┘ │   │
│  └───────────────────────────────┬──────────────────────────────────────┘   │
│                                  │                                          │
│                                  ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Mock Notification Services                         │   │
│  │               (100 instances, echo + metrics only)                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Observability Stack                               │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │ Prometheus  │  │  Grafana    │  │   Jaeger    │  │   Loki      │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Hardware Requirements

| Component | CPU | Memory | Storage | Network |
|-----------|-----|--------|---------|---------|
| Fan-Out Service | 8 cores | 8GB | 50GB SSD | 10Gbps |
| NATS Node (×3) | 4 cores | 8GB | 100GB NVMe | 10Gbps |
| k6 Runner (×4) | 4 cores | 4GB | 20GB | 10Gbps |
| Mock Notif Svc | 2 cores | 2GB | 10GB | 10Gbps |
| Observability | 4 cores | 16GB | 200GB SSD | 1Gbps |

### 4.3 Pre-Population Script

```go
// scripts/populate_routing_table.go
package main

import (
    "fmt"
    "math/rand"
)

type TestDataGenerator struct {
    NumUsers      int
    NumChannels   int
    NumInstances  int
    ChannelsPerUser int
}

func (g *TestDataGenerator) GeneratePresenceEvents() []PresenceEvent {
    events := make([]PresenceEvent, 0, g.NumUsers)

    for userIdx := 0; userIdx < g.NumUsers; userIdx++ {
        userID := fmt.Sprintf("usr_%08d", userIdx)
        instanceID := fmt.Sprintf("notif-%03d", userIdx % g.NumInstances)

        // Assign random channels to user
        channels := g.selectRandomChannels(g.ChannelsPerUser)

        events = append(events, PresenceEvent{
            UserID:     userID,
            InstanceID: instanceID,
            Status:     "online",
            Channels:   channels,
        })
    }

    return events
}

func (g *TestDataGenerator) selectRandomChannels(count int) []string {
    channels := make([]string, count)
    for i := 0; i < count; i++ {
        channelIdx := rand.Intn(g.NumChannels)
        channels[i] = fmt.Sprintf("ch_%08d", channelIdx)
    }
    return channels
}

// Channel size distribution (Zipfian - some channels are very large)
func (g *TestDataGenerator) GenerateChannelDistribution() map[string]int {
    dist := make(map[string]int)

    // 1% of channels have 10K+ members (viral/company-wide)
    // 10% have 100-1000 members (team channels)
    // 89% have 2-100 members (DMs and small groups)

    for i := 0; i < g.NumChannels; i++ {
        channelID := fmt.Sprintf("ch_%08d", i)

        r := rand.Float64()
        switch {
        case r < 0.01:
            dist[channelID] = 10000 + rand.Intn(90000) // 10K-100K
        case r < 0.11:
            dist[channelID] = 100 + rand.Intn(900)     // 100-1000
        default:
            dist[channelID] = 2 + rand.Intn(98)        // 2-100
        }
    }

    return dist
}
```

---

## 5. Test Scenarios

### 5.1 Baseline Load Test

**Purpose:** Establish performance baseline under normal operating conditions.

```
Duration: 30 minutes
Message Rate: 10,000 msg/sec (steady)
Presence Rate: 100 changes/sec
Users: 50,000
Channels: 200,000

Expected Results:
- p99 latency < 5ms
- No consumer lag
- Memory stable
```

### 5.2 Target Load Test

**Purpose:** Validate system meets production requirements.

```
Duration: 2 hours
Message Rate: 50,000 msg/sec (steady)
Presence Rate: 1,000 changes/sec
Users: 100,000
Channels: 500,000

Expected Results:
- p99 latency < 10ms
- Consumer lag < 1,000
- Memory < 3GB
```

### 5.3 Stress Test

**Purpose:** Find breaking point and degradation characteristics.

```
Duration: 1 hour
Message Rate: Ramp 10K → 200K msg/sec over 30 min
Presence Rate: Ramp 100 → 10,000 changes/sec
Users: 500,000
Channels: 2,000,000

Expected Results:
- Identify throughput ceiling
- Characterize degradation curve
- No crashes or data loss
```

### 5.4 Spike Test

**Purpose:** Validate behavior during sudden traffic bursts (viral message).

```
Profile:
  0-5 min:   10,000 msg/sec (baseline)
  5-6 min:   Spike to 150,000 msg/sec (viral event)
  6-10 min:  Decay to 50,000 msg/sec
  10-15 min: Return to 10,000 msg/sec

Spike Characteristics:
- Single channel receives 100,000 messages in 1 minute
- Channel has 50,000 online members across 80 instances

Expected Results:
- System recovers within 2 minutes
- No message loss
- Consumer lag clears within 5 minutes
```

### 5.5 Soak Test

**Purpose:** Identify memory leaks, GC issues, resource exhaustion over time.

```
Duration: 24 hours
Message Rate: 30,000 msg/sec (80% of target)
Presence Rate: 500 changes/sec
Churn: 10% of users disconnect/reconnect every hour

Expected Results:
- Memory stable (no growth trend)
- GC pause times stable
- No degradation in latency over time
```

### 5.6 Chaos Test

**Purpose:** Validate resilience to failures.

```
Scenarios:
1. NATS node failure (1 of 3)
2. Network partition (100ms latency injection)
3. Mock Notification Service failures (10% dropping messages)
4. CPU throttling (limit to 50% for 5 minutes)
5. Memory pressure (allocate 2GB competing process)

Expected Results:
- Graceful degradation
- Automatic recovery
- No data corruption
```

---

## 6. Test Implementation

### 6.1 k6 Message Load Generator

```javascript
// tests/k6/fanout-message-load.js
import { check } from 'k6';
import { Writer } from 'k6/x/nats';

export const options = {
  scenarios: {
    steady_load: {
      executor: 'constant-arrival-rate',
      rate: 50000,           // 50K messages per second
      timeUnit: '1s',
      duration: '2h',
      preAllocatedVUs: 100,
      maxVUs: 500,
    },
  },
  thresholds: {
    'nats_publish_duration': ['p(99)<10'],  // 99th percentile < 10ms
    'checks': ['rate>0.999'],                // 99.9% success rate
  },
};

const natsWriter = new Writer({
  servers: ['nats://nats-1:4222', 'nats://nats-2:4222', 'nats://nats-3:4222'],
  jetstream: true,
});

// Pre-generated channel distribution
const channelWeights = JSON.parse(open('./channel_weights.json'));
const channels = Object.keys(channelWeights);

export default function () {
  // Select channel based on weighted distribution (Zipfian)
  const channelId = selectWeightedChannel(channels, channelWeights);

  const message = {
    event_id: `evt_${__VU}_${__ITER}_${Date.now()}`,
    type: 'message.sent',
    channel_id: channelId,
    sender_id: `usr_${__VU % 100000}`,
    content: {
      text: 'Performance test message',
      timestamp: new Date().toISOString(),
    },
    metadata: {
      channel_member_count: channelWeights[channelId],
    },
  };

  const startTime = Date.now();

  const result = natsWriter.publish(
    `messages.send.${channelId}`,
    JSON.stringify(message),
    {
      msgId: message.event_id,  // Deduplication
    }
  );

  const duration = Date.now() - startTime;

  check(result, {
    'publish succeeded': (r) => r.success,
    'latency under 10ms': () => duration < 10,
  });
}

function selectWeightedChannel(channels, weights) {
  // Zipfian distribution - larger channels get more traffic
  const totalWeight = Object.values(weights).reduce((a, b) => a + b, 0);
  let random = Math.random() * totalWeight;

  for (const [channel, weight] of Object.entries(weights)) {
    random -= weight;
    if (random <= 0) {
      return channel;
    }
  }

  return channels[0];
}

export function teardown() {
  natsWriter.close();
}
```

### 6.2 k6 Presence Churn Generator

```javascript
// tests/k6/fanout-presence-churn.js
import { check, sleep } from 'k6';
import redis from 'k6/x/redis';

export const options = {
  scenarios: {
    presence_churn: {
      executor: 'constant-arrival-rate',
      rate: 1000,            // 1000 presence changes per second
      timeUnit: '1s',
      duration: '2h',
      preAllocatedVUs: 50,
      maxVUs: 200,
    },
  },
  thresholds: {
    'redis_command_duration': ['p(99)<50'],
    'checks': ['rate>0.99'],
  },
};

const redisClient = new redis.Client({
  addrs: ['redis:6379'],
});

const NUM_USERS = 100000;
const NUM_INSTANCES = 100;

export default function () {
  const userId = `usr_${Math.floor(Math.random() * NUM_USERS).toString().padStart(8, '0')}`;
  const instanceId = `notif-${(Math.floor(Math.random() * NUM_INSTANCES)).toString().padStart(3, '0')}`;

  // Simulate user coming online or going offline
  const isOnline = Math.random() > 0.3;  // 70% online, 30% offline

  const startTime = Date.now();

  if (isOnline) {
    // User comes online
    redisClient.hset(`presence:user:${userId}`, {
      status: 'online',
      instance_id: instanceId,
      connected_at: new Date().toISOString(),
    });
    redisClient.expire(`presence:user:${userId}`, 120);
    redisClient.publish('presence:changes', `${userId}:online:${instanceId}`);
  } else {
    // User goes offline
    redisClient.hset(`presence:user:${userId}`, {
      status: 'offline',
      last_seen: new Date().toISOString(),
    });
    redisClient.publish('presence:changes', `${userId}:offline`);
  }

  const duration = Date.now() - startTime;

  check(true, {
    'presence update succeeded': () => true,
    'latency under 50ms': () => duration < 50,
  });
}
```

### 6.3 k6 Spike Test

```javascript
// tests/k6/fanout-spike-test.js
import { check } from 'k6';
import { Writer } from 'k6/x/nats';

export const options = {
  scenarios: {
    spike_test: {
      executor: 'ramping-arrival-rate',
      startRate: 10000,
      timeUnit: '1s',
      preAllocatedVUs: 200,
      maxVUs: 1000,
      stages: [
        { duration: '5m', target: 10000 },   // Baseline
        { duration: '1m', target: 150000 },  // Spike!
        { duration: '4m', target: 50000 },   // Elevated
        { duration: '5m', target: 10000 },   // Recovery
      ],
    },
  },
  thresholds: {
    'nats_publish_duration': ['p(99)<100'],  // Allow higher latency during spike
    'checks': ['rate>0.99'],
  },
};

// Viral channel - 50K members across 80 instances
const VIRAL_CHANNEL = 'ch_viral_event';

const natsWriter = new Writer({
  servers: ['nats://nats-1:4222'],
  jetstream: true,
});

export default function () {
  // During spike, 80% of traffic goes to viral channel
  const isViralTraffic = Math.random() < 0.8;
  const channelId = isViralTraffic ? VIRAL_CHANNEL : `ch_${Math.floor(Math.random() * 500000).toString().padStart(8, '0')}`;

  const message = {
    event_id: `evt_spike_${__VU}_${__ITER}_${Date.now()}`,
    type: 'message.sent',
    channel_id: channelId,
    sender_id: `usr_${__VU % 100000}`,
    content: {
      text: isViralTraffic ? '🚀 VIRAL MESSAGE!' : 'Normal message',
    },
    metadata: {
      channel_member_count: isViralTraffic ? 50000 : 100,
    },
  };

  const result = natsWriter.publish(
    `messages.send.${channelId}`,
    JSON.stringify(message),
    { msgId: message.event_id }
  );

  check(result, {
    'publish succeeded': (r) => r.success,
  });
}
```

### 6.4 End-to-End Latency Measurement

```javascript
// tests/k6/fanout-e2e-latency.js
import { check } from 'k6';
import { Writer, Reader } from 'k6/x/nats';
import { Trend, Counter } from 'k6/metrics';

const e2eLatency = new Trend('fanout_e2e_latency_ms');
const messagesReceived = new Counter('messages_received');

export const options = {
  scenarios: {
    e2e_latency: {
      executor: 'constant-arrival-rate',
      rate: 1000,  // Lower rate for accurate latency measurement
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 20,
      maxVUs: 50,
    },
  },
  thresholds: {
    'fanout_e2e_latency_ms': ['p(50)<5', 'p(99)<20', 'p(99.9)<100'],
  },
};

const natsWriter = new Writer({
  servers: ['nats://nats-1:4222'],
  jetstream: true,
});

// Subscribe to a mock notification service inbox
const natsReader = new Reader({
  servers: ['nats://nats-1:4222'],
  subject: 'instance.events.notif-test',
});

const pendingMessages = new Map();

export default function () {
  const messageId = `latency_${__VU}_${__ITER}_${Date.now()}`;
  const sendTime = Date.now();

  // Track this message
  pendingMessages.set(messageId, sendTime);

  const message = {
    event_id: messageId,
    type: 'message.sent',
    channel_id: 'ch_latency_test',  // Channel routed to notif-test instance
    sender_id: 'usr_latency_test',
    content: { text: 'Latency test' },
    _send_timestamp: sendTime,
  };

  natsWriter.publish(
    'messages.send.ch_latency_test',
    JSON.stringify(message),
    { msgId: messageId }
  );
}

// Background goroutine to receive messages
export function setup() {
  // Start receiver in background
  natsReader.subscribe((msg) => {
    const data = JSON.parse(msg.data);
    const receiveTime = Date.now();

    if (pendingMessages.has(data.event_id)) {
      const sendTime = pendingMessages.get(data.event_id);
      const latency = receiveTime - sendTime;

      e2eLatency.add(latency);
      messagesReceived.add(1);

      pendingMessages.delete(data.event_id);
    }
  });
}
```

---

## 7. Monitoring & Observability

### 7.1 Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Fan-Out Service Performance",
    "panels": [
      {
        "title": "Message Throughput",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(fanout_messages_processed_total[1m])",
            "legendFormat": "Processed/sec"
          },
          {
            "expr": "rate(fanout_messages_routed_total[1m])",
            "legendFormat": "Routed/sec (by instance)"
          }
        ]
      },
      {
        "title": "Routing Latency",
        "type": "heatmap",
        "targets": [
          {
            "expr": "rate(fanout_routing_latency_seconds_bucket[1m])",
            "legendFormat": "{{le}}"
          }
        ]
      },
      {
        "title": "Consumer Lag",
        "type": "graph",
        "targets": [
          {
            "expr": "fanout_jetstream_consumer_lag",
            "legendFormat": "{{stream}}/{{consumer}}"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": { "params": [10000], "type": "gt" },
              "query": { "params": ["A", "5m", "now"] }
            }
          ]
        }
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "process_resident_memory_bytes{job='fanout'}",
            "legendFormat": "RSS"
          },
          {
            "expr": "fanout_routing_table_bytes",
            "legendFormat": "Routing Table"
          },
          {
            "expr": "go_memstats_heap_inuse_bytes{job='fanout'}",
            "legendFormat": "Heap In-Use"
          }
        ]
      },
      {
        "title": "GC Pause Times",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(go_gc_pause_seconds_total{job='fanout'}[1m])",
            "legendFormat": "GC pause/sec"
          },
          {
            "expr": "histogram_quantile(0.99, rate(go_gc_pause_seconds_bucket{job='fanout'}[5m]))",
            "legendFormat": "p99 GC pause"
          }
        ]
      },
      {
        "title": "Lock Contention",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(fanout_rwmutex_wait_seconds_bucket[1m]))",
            "legendFormat": "p99 {{lock_type}} lock wait"
          }
        ]
      },
      {
        "title": "Publishes per Message",
        "type": "histogram",
        "targets": [
          {
            "expr": "histogram_quantile(0.5, rate(fanout_publishes_per_message_bucket[5m]))",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.99, rate(fanout_publishes_per_message_bucket[5m]))",
            "legendFormat": "p99"
          }
        ]
      },
      {
        "title": "Presence Updates",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(fanout_presence_updates_total[1m])",
            "legendFormat": "Updates/sec"
          },
          {
            "expr": "histogram_quantile(0.99, rate(fanout_presence_update_latency_seconds_bucket[1m]))",
            "legendFormat": "p99 latency"
          }
        ]
      }
    ]
  }
}
```

### 7.2 Alerting Rules

```yaml
# prometheus/alerts/fanout.yml
groups:
  - name: fanout-performance
    rules:
      - alert: FanOutHighLatency
        expr: histogram_quantile(0.99, rate(fanout_routing_latency_seconds_bucket[5m])) > 0.02
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Fan-Out p99 latency > 20ms"
          description: "Fan-Out routing latency is {{ $value | humanizeDuration }}"

      - alert: FanOutConsumerLagCritical
        expr: fanout_jetstream_consumer_lag > 10000
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Fan-Out consumer lag > 10K messages"
          description: "Consumer lag is {{ $value }} messages"

      - alert: FanOutMemoryHigh
        expr: process_resident_memory_bytes{job="fanout"} > 4e9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Fan-Out memory > 4GB"
          description: "Memory usage is {{ $value | humanize1024 }}"

      - alert: FanOutGCPauseLong
        expr: histogram_quantile(0.99, rate(go_gc_pause_seconds_bucket{job="fanout"}[5m])) > 0.05
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Fan-Out GC pauses > 50ms"
          description: "p99 GC pause is {{ $value | humanizeDuration }}"
```

---

## 8. Success Criteria

### 8.1 Test Pass/Fail Criteria

| Test Type | Pass Criteria | Fail Criteria |
|-----------|--------------|---------------|
| **Baseline** | All SLOs met for 30 min | Any SLO violation |
| **Target** | All SLOs met for 2 hours | >1% SLO violations |
| **Stress** | Graceful degradation, no crashes | Crash, data loss, or unrecoverable state |
| **Spike** | Recovery within 5 min | Consumer lag doesn't clear in 10 min |
| **Soak** | No resource growth trend | >10% memory growth over 24 hours |
| **Chaos** | Auto-recovery from all scenarios | Manual intervention required |

### 8.2 Performance Regression Gates

```yaml
# CI/CD performance gates
performance_gates:
  # Block deployment if:
  regression_thresholds:
    latency_p99_increase: 20%      # p99 latency increased by >20%
    throughput_decrease: 10%        # Max throughput decreased by >10%
    memory_increase: 15%            # Memory usage increased by >15%

  # Require investigation if:
  warning_thresholds:
    latency_p99_increase: 10%
    throughput_decrease: 5%
    memory_increase: 10%
```

---

## 9. Bottleneck Analysis

### 9.1 Common Bottlenecks

| Symptom | Likely Cause | Investigation | Mitigation |
|---------|--------------|---------------|------------|
| High p99 latency, low CPU | Lock contention | `fanout_rwmutex_wait_seconds` | Shard routing table |
| Consumer lag growing | Publishing slower than consuming | NATS connection metrics | Add workers, batch larger |
| Memory growing over time | Routing table leak | Heap profile | Fix user eviction |
| Latency spikes every N seconds | GC pauses | `go_gc_pause_seconds` | Tune GOGC, reduce allocations |
| High CPU, low throughput | Hot channel (many members) | `fanout_publishes_per_message` | Parallelize large fan-outs |

### 9.2 Profiling Commands

```bash
# CPU profile during load test
curl -o cpu.pprof http://fanout:6060/debug/pprof/profile?seconds=30
go tool pprof -http=:8080 cpu.pprof

# Memory profile
curl -o heap.pprof http://fanout:6060/debug/pprof/heap
go tool pprof -http=:8080 heap.pprof

# Goroutine profile (for lock contention)
curl -o goroutine.pprof http://fanout:6060/debug/pprof/goroutine
go tool pprof -http=:8080 goroutine.pprof

# Block profile (for lock waits)
curl -o block.pprof http://fanout:6060/debug/pprof/block
go tool pprof -http=:8080 block.pprof

# Mutex profile
curl -o mutex.pprof http://fanout:6060/debug/pprof/mutex
go tool pprof -http=:8080 mutex.pprof
```

### 9.3 Routing Table Analysis

```go
// Debug endpoint: /debug/routing-stats
func (s *FanOutService) HandleRoutingStats(w http.ResponseWriter, r *http.Request) {
    s.channelIndex.mu.RLock()
    defer s.channelIndex.mu.RUnlock()

    stats := RoutingStats{
        TotalChannels:      len(s.channelIndex.channels),
        TotalUsers:         len(s.userIndex.users),
        ChannelsBySize:     make(map[string]int),
        TopChannels:        make([]ChannelInfo, 0, 10),
        InstanceDistribution: make(map[string]int),
    }

    // Analyze channel size distribution
    for channelID, instances := range s.channelIndex.channels {
        totalUsers := 0
        for _, entry := range instances {
            totalUsers += len(entry.UserIDs)
            stats.InstanceDistribution[entry.InstanceID]++
        }

        // Bucket by size
        switch {
        case totalUsers > 10000:
            stats.ChannelsBySize["10K+"]++
        case totalUsers > 1000:
            stats.ChannelsBySize["1K-10K"]++
        case totalUsers > 100:
            stats.ChannelsBySize["100-1K"]++
        default:
            stats.ChannelsBySize["<100"]++
        }

        // Track top channels
        if totalUsers > 1000 {
            stats.TopChannels = append(stats.TopChannels, ChannelInfo{
                ID:          channelID,
                TotalUsers:  totalUsers,
                NumInstances: len(instances),
            })
        }
    }

    json.NewEncoder(w).Encode(stats)
}
```

---

## 10. Performance Runbook

### 10.1 High Latency Investigation

```
SYMPTOM: fanout_routing_latency_seconds p99 > 20ms

STEPS:
1. Check consumer lag
   $ nats consumer info MESSAGES fan-out-pool
   → If lag > 1000: proceed to "Consumer Lag" runbook

2. Check lock contention
   $ curl http://fanout:6060/debug/routing-stats | jq .
   → High read lock wait: too many presence updates
   → High write lock wait: routing table too large

3. Check for hot channels
   $ curl http://fanout:6060/debug/routing-stats | jq '.TopChannels'
   → If single channel has >50K users: consider channel sharding

4. Profile CPU
   $ curl -o cpu.pprof http://fanout:6060/debug/pprof/profile?seconds=30
   $ go tool pprof -top cpu.pprof
   → Look for unexpected hot functions

5. Check GC
   $ curl http://fanout:9090/metrics | grep go_gc
   → If gc_pause_seconds_total increasing: tune GOGC or reduce allocations
```

### 10.2 Consumer Lag Investigation

```
SYMPTOM: fanout_jetstream_consumer_lag > 10000

STEPS:
1. Check NATS cluster health
   $ nats server check
   → Any node unhealthy: failover or restart

2. Check Fan-Out worker health
   $ kubectl get pods -l app=fanout
   → If pods restarting: check OOM kills, logs

3. Check publish rate
   $ curl http://fanout:9090/metrics | grep fanout_messages_routed
   → If rate << input rate: workers not keeping up

4. Check batch processing time
   $ curl http://fanout:9090/metrics | grep fanout_batch_processing_latency
   → If batch processing > 100ms: investigate slow publishes

5. Scale Fan-Out workers
   $ kubectl scale deployment fanout --replicas=5
   → Monitor lag decrease

6. If still lagging: check NATS publish latency
   $ nats latency --server-b nats://nats-1:4222 --duration 10s
   → High latency: network or NATS issue
```

### 10.3 Memory Growth Investigation

```
SYMPTOM: Memory usage steadily increasing over hours

STEPS:
1. Compare routing table size to expected
   $ curl http://fanout:6060/debug/routing-stats | jq '.TotalUsers, .TotalChannels'
   → Expected: ~100K users, ~500K channels
   → If much higher: presence eviction not working

2. Check for zombie users (offline but still in table)
   $ curl http://fanout:6060/debug/routing-stats | jq '.ZombieUsers'
   → If > 0: fix presence TTL handling

3. Take heap profile
   $ curl -o heap1.pprof http://fanout:6060/debug/pprof/heap
   $ # Wait 1 hour
   $ curl -o heap2.pprof http://fanout:6060/debug/pprof/heap
   $ go tool pprof -base heap1.pprof heap2.pprof
   → Identify growing allocations

4. Check membership cache
   $ curl http://fanout:6060/debug/cache-stats | jq .
   → If cache size > expected: LRU eviction broken

5. Force GC and compare
   $ curl -X POST http://fanout:6060/debug/gc
   $ curl http://fanout:9090/metrics | grep process_resident_memory
   → If memory drops significantly: GC tuning issue
```

---

## Related Documents

- [Detailed Design](../detailed-design.md) — Fan-Out Service specifications
- [Operations Guide](../operations/operations-guide.md) — Operational runbooks
- [Sequence Diagrams](../diagrams/sequence-diagrams.md) — Fan-Out Service flows
