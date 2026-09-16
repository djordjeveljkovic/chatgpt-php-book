# AI Summary — Chapter 204 — Application Services

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Explains application services as use-case coordinators, typed commands and results, authorization context, transaction and external-effect boundaries, idempotency, error mapping, testing, operations, exercises, and review questions.

## Concepts already explained

Application services coordinate domain objects, repositories, transactions, events, and ports. They should not own domain rules or hold slow external calls inside local transactions. Concurrent idempotency requires a database or durable uniqueness boundary.

## Terminology established

Application service, use-case handler, command, result model, transaction boundary, idempotency scope, outbox.

## Examples used

A `PlaceOrder` command, receipt result, and handler coordinating authorization, repository persistence, idempotency, and events.

## Cross-references

- [Chapter 202 — Domain Services](../../volumes/13-architecture/202-domain-services.md)
- [Chapter 203 — Domain Events](../../volumes/13-architecture/203-domain-events.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 205 — Event-Driven Architecture: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XIII proofread. Application-service guidance links to Fowler and PHP documentation.
