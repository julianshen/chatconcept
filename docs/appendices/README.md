# Appendices

Supporting documentation for the Communication Platform architecture.

## Contents

| Document | Description |
|----------|-------------|
| [Implementation Plan](./implementation-plan.md) | Phased rollout with milestones and team allocation |
| [Risks and Mitigations](./risks.md) | Risk register with likelihood, impact, and mitigations |
| [Open Questions](./open-questions.md) | Unresolved design questions and decision log |

## Overview

### Implementation Plan

Six-phase implementation strategy:

| Phase | Focus | Duration |
|-------|-------|----------|
| 0 | Foundation (NATS, Cassandra, observability) | 2-3 weeks |
| 1 | Command path (write flow) | 3-4 weeks |
| 2 | Query path (read flow) | 2-3 weeks |
| 3 | Fan-out & notification (real-time) | 4-5 weeks |
| 4 | Workers & webhooks | 3-4 weeks |
| 5-6 | Migration & decommission | 4-5 weeks |

### Risk Register

Categorized risks with mitigation strategies:

- **Infrastructure:** NATS cluster failure, Cassandra hotspots, GC pauses
- **Application:** Routing table corruption, cold start delays, eventual consistency
- **Operational:** Complexity increase, protocol migration
- **Security:** Webhook SSRF, delivery backlog

### Open Questions

Tracks design questions in three states:

1. **Resolved** — Addressed in ADRs or feature documents
2. **Open** — Needs design decision
3. **Deferred** — Not required for initial release
