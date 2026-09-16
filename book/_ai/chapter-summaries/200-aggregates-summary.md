# AI Summary — Chapter 200 — Aggregates

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Aggregates define consistency boundaries around roots, entities, and value objects. The chapter covers root commands, invariant scope, concurrency, size, read models, events, reconstitution, deletion, testing, and operations.

## Concepts already explained

Aggregate root, consistency boundary, invariant, command, projection, optimistic version, lock, bounded collection, outbox event, and eventual consistency.

## Examples used

A PHP Order aggregate, order lines, confirmation rules, and versioned persistence.

## Cross-references

- [Chapter 197 — Domain-Driven Design](../../volumes/13-architecture/197-domain-driven-design.md)
- [Chapter 198 — Entities](../../volumes/13-architecture/198-entities.md)
- [Chapter 201 — Repositories](../../volumes/13-architecture/201-repositories.md)
- [Chapter 136 — Webhooks](../../volumes/09-http-and-application-development/136-webhooks.md)

## Open threads

Continue with Chapter 201 on persistence contracts around aggregates and read models.

## Exact next section

Chapter 201 — Repositories: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; live database and queue tests were not run.
