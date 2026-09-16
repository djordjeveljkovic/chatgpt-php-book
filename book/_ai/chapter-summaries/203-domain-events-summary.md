# AI Summary — Chapter 203 — Domain Events

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Explains immutable domain facts, event identity, aggregate event collection, transactional outbox, integration-event mapping, commands versus events, schema evolution, idempotent consumers, testing, operations, exercises, and review questions.

## Concepts already explained

Events state that something happened; they are not automatically commands or reliable delivery. Persist aggregate state and outbox data atomically, expect at-least-once publication, version public schemas, and deduplicate consumers.

## Terminology established

Domain event, integration event, event ID, outbox, dual-write gap, idempotent consumer, event version.

## Examples used

An immutable `OrderPlaced` event, aggregate event collection, outbox boundary, and integration-event mapping guidance.

## Cross-references

- [Chapter 202 — Domain Services](../../volumes/13-architecture/202-domain-services.md)
- [Chapter 205 — Event-Driven Architecture](../../volumes/13-architecture/205-event-driven-architecture.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 204 — Application Services: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XIII proofread. Event guidance links to Fowler and Enterprise Integration Patterns.
