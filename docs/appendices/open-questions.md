# Open Questions

This document tracks unresolved design questions and areas requiring further investigation.

**Related Documents:**
- [Architecture Overview](../overview.md) — High-level architecture
- [Risks and Mitigations](./risks.md) — Risk register
- [ADR Index](../adrs/README.md) — Architecture decisions

---

## Resolved Questions

These questions have been addressed in subsequent design documents.

### Thread Support

> How are threaded messages modeled in the event stream and Cassandra partition design?

**Resolution:** See [ADR-006: Thread Storage](../adrs/ADR-006-thread-storage.md) and [Thread Support Feature](../features/threads.md).

Dual-write strategy with separate `thread_messages` table partitioned by `(thread_id, bucket)`. Messages with `also_send_to_channel: true` are written to both tables.

---

### Read Receipts / Delivery Receipts

> Should delivery confirmation flow back from Notification Service → Fan-Out Service → Cassandra? This creates a write amplification concern.

**Resolution:** See [ADR-008: Hybrid Read Receipts](../adrs/ADR-008-hybrid-receipts.md) and [Receipts Feature](../features/receipts.md).

Hybrid approach: per-message receipts for channels ≤50 members, aggregate last-read pointers for larger channels. This bounds write amplification at ~10K writes/sec vs. 750K+ for full receipts.

---

### Client Reconnection Protocol

> When a client reconnects to a different Notification Service instance, how does it determine which messages it missed?

**Resolution:** See [ADR-007: Tiered Client Reconnection](../adrs/ADR-007-tiered-reconnection.md) and [Reconnection Feature](../features/reconnection.md).

Four-tier catchup strategy based on gap duration:
- < 2 min: JetStream replay
- 2 min - 1 hr: Cassandra active channels
- 1 hr - 24 hr: NATS KV unread counts
- \> 24 hr: Full REST refresh

---

### Fan-Out Service Partitioning (Extreme Scale)

> At >500K simultaneous active channels, should the Fan-Out Service's routing table be sharded across workers?

**Resolution:** See [ADR-009: Fan-Out Consistent Hash Partitioning](../adrs/ADR-009-fanout-partitioning.md) and [Extreme Scale Partitioning Feature](../features/extreme-scale-partitioning.md).

Jump consistent hashing with 256 fixed partitions. Workers assigned contiguous ranges. Activated only when >500K online users.

---

## Open Questions

### File Uploads

> Should file metadata events flow through NATS, or should uploads use a direct-to-object-storage flow with only a reference event published?

**Options:**
1. **Full event flow:** Upload metadata → NATS → Message Writer → Cassandra
2. **Direct upload:** Client → S3 presigned URL → Reference event only

**Considerations:**
- Direct upload reduces latency for large files
- Event flow maintains consistency with message processing
- Hybrid: presigned upload + reference event after upload complete

**Status:** Needs design decision

---

### Federation

> If multi-server federation is required, how do NATS super-clusters map to federated instances?

**Considerations:**
- NATS super-clusters support geo-distribution
- Federation requires event routing between independent deployments
- Identity management across federated instances
- Message routing across federation boundaries

**Questions to resolve:**
- Is this Matrix-style open federation or closed enterprise federation?
- What events need to cross federation boundaries?
- How are user identities resolved across instances?

**Status:** Deferred — not required for initial release

---

### E2E Encryption

> How do end-to-end encrypted messages interact with the Search Indexer worker?

**Constraints:**
- Server cannot read E2E encrypted content
- Search indexer cannot index encrypted text
- Metadata (sender, timestamp, channel) remains searchable

**Options:**
1. **No server-side search for E2E:** Client-side search only
2. **Encrypted search index:** Searchable encryption schemes (complex, performance impact)
3. **Metadata-only search:** Search by sender, date, channel but not content

**Questions to resolve:**
- What percentage of messages will be E2E encrypted?
- Is client-side search acceptable for E2E channels?
- Should E2E be per-channel or per-message?

**Status:** Needs requirements clarification

---

### Message Retention Policies

> Per-channel retention policies may require custom Cassandra TTL management beyond the global stream retention.

**Considerations:**
- NATS JetStream has global stream retention (30 days default)
- Cassandra can have per-row TTL
- Different channels may have different compliance requirements
- Regulatory requirements (GDPR, HIPAA) may mandate deletion

**Questions to resolve:**
- Should retention be configurable per-channel?
- How to handle retention policy changes (backfill deletions)?
- What happens to thread replies when parent is deleted?
- How does retention interact with search index?

**Status:** Needs product requirements

---

### Webhook Idempotency

> Should outgoing webhooks include a delivery ID for bot-side deduplication?

**Current design:** Events include `event_id` which is unique per event but not per delivery attempt.

**Considerations:**
- Retry logic may deliver same event multiple times
- Bot servers should be idempotent but may not be
- Adding `delivery_id` helps bot-side deduplication
- Could also add `attempt_number` for debugging

**Recommendation:** Add both `delivery_id` (UUID per attempt) and `attempt_number` to webhook payload.

**Status:** Minor enhancement, implement in Phase 4

---

### Webhook Event Replay

> Should the platform offer a "replay recent events" API for webhooks that missed events during downtime?

**Current design:** No explicit replay API. Events older than 7 days are lost.

**Options:**
1. **No replay:** Webhook owners responsible for handling missed events
2. **Manual replay endpoint:** Admin API to replay date range to specific webhook
3. **Self-service replay:** Webhook owner can request replay via API

**Considerations:**
- Replay could cause duplicate processing if not idempotent
- Storage implications for keeping replay-able events
- Rate limiting needed to prevent abuse

**Status:** Deferred — evaluate based on customer feedback

---

### Slash Commands via Webhooks

> Should slash commands (e.g., `/deploy staging`) be implemented as a webhook variant?

**Proposed flow:**
1. User types `/deploy staging` in channel
2. Platform routes to registered bot for `/deploy` command
3. Bot receives command payload via webhook
4. Bot processes and posts response back to channel

**Questions to resolve:**
- How are slash commands registered and discovered?
- What's the response timeout for interactive commands?
- Can slash commands have auto-complete suggestions?
- How does authentication work for command handlers?

**Status:** Feature request — prioritize after core webhooks stable

---

## Decision Log

| Question | Date Raised | Date Resolved | Resolution |
|----------|-------------|---------------|------------|
| Thread support | 2026-02-01 | 2026-02-01 | ADR-006 |
| Read receipts | 2026-02-01 | 2026-02-01 | ADR-008 |
| Client reconnection | 2026-02-01 | 2026-02-01 | ADR-007 |
| Fan-out partitioning | 2026-02-01 | 2026-02-01 | ADR-009 |
| File uploads | 2026-02-01 | — | Open |
| Federation | 2026-02-01 | — | Deferred |
| E2E encryption | 2026-02-01 | — | Open |
| Message retention | 2026-02-01 | — | Open |
| Webhook idempotency | 2026-02-01 | — | Open |
| Webhook replay | 2026-02-01 | — | Deferred |
| Slash commands | 2026-02-01 | — | Deferred |
