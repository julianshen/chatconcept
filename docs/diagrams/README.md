# Architecture Diagrams

Visual representations of the Communication Platform architecture.

## Contents

| Document | Description |
|----------|-------------|
| [C4 Diagrams](./c4-diagrams.md) | System Context, Container, and Component diagrams |
| [Sequence Diagrams](./sequence-diagrams.md) | Message flows and interaction patterns |

## Diagram Types

### C4 Model Diagrams

The [C4 model](https://c4model.com/) provides a hierarchical set of abstractions:

1. **System Context** — Shows the platform and its external dependencies
2. **Container** — Shows major services, databases, and message brokers
3. **Component** — Shows internal structure of key services (Fan-Out, Notification)

### Sequence Diagrams

Key flows through the system:

- **Send Message** — Write path from client to persistence and fan-out
- **User Connection** — WebSocket connection and presence updates
- **Typing Indicator** — Ephemeral event routing
- **Thread Reply** — Dual-write strategy for threads
- **Reconnection** — Tiered catchup protocol
- **Read Receipt** — Hybrid receipt handling
- **Fan-Out Service Internal** — Detailed internal processing flows:
  - Message event processing with batched publish
  - Presence change processing (online/offline)
  - Membership change processing (join/leave)
  - Typing indicator routing

## Rendering

All diagrams use [Mermaid](https://mermaid.js.org/) syntax and can be rendered by:

- GitHub markdown preview
- VS Code with Mermaid extension
- Mermaid Live Editor (https://mermaid.live)
- Documentation sites (Docusaurus, MkDocs with plugins)
