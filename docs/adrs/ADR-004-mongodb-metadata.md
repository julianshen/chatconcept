# ADR-004: Retain MongoDB for Metadata Only

**Date:** 2026-02-01
**Status:** Accepted
**Deciders:** Architecture Team

## Context

MongoDB is deeply embedded in the ecosystem for all data storage. A full migration away from MongoDB would be risky and unnecessary. However, using MongoDB for _all_ data (including high-volume message history) creates scaling bottlenecks.

## Decision

Retain MongoDB exclusively for **metadata**: Users, Channels, Permissions, Settings, Roles, OAuth tokens, webhook registrations, and similar low-to-medium volume, schema-flexible data.

Message history moves to Cassandra.

## Rationale

### Why Keep MongoDB for Metadata?

- **Flexible schema:** MongoDB's document model handles diverse and evolving metadata structures well
- **Rich query language:** Aggregation pipeline is powerful for complex metadata queries
- **Mature ecosystem:** Existing admin tools, migration scripts, and backup procedures continue to work
- **Low volume:** Metadata write volume is orders of magnitude lower than message volume

### Separation of Concerns

| Data Type | Storage | Rationale |
|-----------|---------|-----------|
| Messages | Cassandra | Write-optimized, time-series partitioning |
| Users, Channels, Permissions | MongoDB | Flexible schema, rich queries |
| Search Index | Elasticsearch | Full-text indexing |
| Ephemeral State | Redis/Valkey | Presence, typing, routing metadata |

### Operational Simplicity

A single MongoDB replica set handles metadata workloads comfortably. No need for MongoDB sharding, which adds significant operational complexity.

## Consequences

**Positive:**

- Minimal disruption to existing metadata-related code and tooling
- MongoDB replica set is easy to operate and back up
- Clear separation of concerns: metadata = MongoDB, messages = Cassandra

**Negative:**

- Two database technologies to operate. Mitigation: Clearly defined ownership boundaries; metadata is low-ops.
- Some queries that previously joined messages and metadata in MongoDB now require cross-database operations. Mitigation: The Query Service handles this at the application layer; denormalize sender display names into message events.
