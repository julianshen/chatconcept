# ADR-008: Hybrid Read Receipts — Per-Message for Small Groups, Aggregate for Large Channels

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

Users expect "read by" indicators in DMs and small groups (like WhatsApp/iMessage checkmarks) but not in large channels (Slack-like behavior). Implementing per-message receipts for all channels creates unsustainable write amplification.

### Write Amplification Analysis

| Metric | Value |
|--------|-------|
| Global message rate | 50,000 msg/sec |
| Average channel size | ~50 members |
| Average read ratio | ~30% of members |
| **Per-message receipt writes** | 50K × 50 × 0.30 = **750K writes/sec** |

At 750K receipt writes/sec, the receipt table would generate **more write volume than the message table itself**.

## Decision

Use a **hybrid approach**:
- Per-message receipts for channels with ≤ 50 members
- Aggregate last-read pointers for channels with > 50 members

## Rationale

| Channel Type | Members | Per-Message Write Volume | Aggregate Write Volume |
|--------------|---------|-------------------------|----------------------|
| DM | 2 | 2 receipts/message | 1 pointer update/visit |
| Small group | 5–50 | 50 receipts/message | 1 pointer update/visit |
| Large channel | 100–100K | 100–100K receipts/message ❌ | 1 pointer update/visit ✅ |

### Why 50 Members?

1. Groups ≤ 50 members commonly show "read by" UI (WhatsApp, Teams, Slack DMs)
2. At 50 members, receipt write amplification is 50x — manageable
3. Above 50 members, "read by" UI is less useful (too many names) and write cost is prohibitive

### Implementation

The receipt writer checks `channel_member_count` (denormalized in the message event) and only persists per-message receipts below the threshold.

## Consequences

**Positive:**

- "Read by" UI available for DMs and small groups
- Write volume bounded at ~10K receipt writes/sec (vs. 750K+ for full receipts)
- Large channel unread tracking is nearly free (KV pointer updates)

**Negative:**

- No per-message "read by" UI for large channels
- The 50-member threshold is somewhat arbitrary — could be made configurable per channel
- Channel membership changes may cross the threshold, requiring migration logic
