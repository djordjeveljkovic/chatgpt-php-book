# AI Summary — Chapter 205 — Event-Driven Architecture

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Explains event-driven communication, commands versus events, delivery guarantees, idempotent consumers, outbox and inbox patterns, choreography and orchestration, schema evolution, testing, operations, exercises, and review questions.

## Concepts already explained

Event-driven architecture reduces direct coupling while introducing delivery, duplication, ordering, schema, and observability concerns. At-least-once delivery requires idempotency; in-memory fakes cannot prove broker acknowledgment and offset behavior.

## Terminology established

Event-driven architecture, producer, consumer, topic, stream, at-least-once delivery, inbox, dead letter, choreography, orchestration.

## Examples used

An idempotent `OrderPlaced` consumer, outbox/inbox boundaries, workflow comparison, and event-schema guidance.

## Cross-references

- [Chapter 203 — Domain Events](../../volumes/13-architecture/203-domain-events.md)
- [Chapter 206 — Distributed Systems](../../volumes/13-architecture/206-distributed-systems.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 206 — Distributed Systems: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XIII proofread. Event guidance links to Fowler, microservices.io, and Enterprise Integration Patterns.
