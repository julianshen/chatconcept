# ADR-006: Thread Storage — Separate Table with Dual Write

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

Threads are a core feature of modern chat platforms. Thread replies need to be queryable both:
1. As part of the channel timeline (if "also sent to channel")
2. As a standalone thread history

The existing `messages` table is partitioned by `(channel_id, bucket)` — querying "all replies to thread X" across daily buckets is expensive without a secondary index.

## Options Considered

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: `thread_id` column in `messages`** | Add nullable column, use `ALLOW FILTERING` or materialized view | Simpler, single write path | `ALLOW FILTERING` is full-partition scan; MVs are fragile |
| **B: Separate `thread_messages` table** | Dedicated table partitioned by `(thread_id, bucket)`. Dual write. | Optimal read path, clean partition design | Dual write complexity, storage duplication |
| **C: Thread replies only in `thread_messages`** | Don't write to `messages` at all for replies | Single write, no duplication | "Also send to channel" requires querying both tables |

## Decision

**Option B** — Separate `thread_messages` table with dual write.

## Rationale

Thread queries (`getThreadMessages`) need to efficiently scan all replies to a single thread in chronological order. Partitioning by `(thread_id, bucket)` provides exactly this access pattern.

The `messages` table remains the single source of truth for the complete channel timeline, preserving existing query paths.

### Dual Write Strategy

When a thread reply arrives, the Message Writer performs:

1. **Always:** Write to `thread_messages` table (for thread panel queries)
2. **If `also_send_to_channel: true`:** Write to `messages` table (for channel timeline)
3. **Always:** Update `thread_summary` (increment reply_count)

Both writes target different Cassandra partitions and execute in parallel. The JetStream acknowledgment is only sent after both writes succeed.

### Storage Duplication

Only messages with `also_send_to_channel: true` are written to both tables. Most thread replies (~80% in Slack's published data) are thread-only.

## Consequences

**Positive:**

- Optimal read performance for both channel timeline and thread history
- Independent compaction and TTL management per table
- Clean Cassandra partition design without secondary indexes

**Negative:**

- Message Writer has dual-write complexity
- "Also send to channel" replies consume ~2x storage
- Thread reply counts in `thread_summary` require CAS updates (lightweight transactions)
