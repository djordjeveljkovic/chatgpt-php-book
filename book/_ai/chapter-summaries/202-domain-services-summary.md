# AI Summary — Chapter 202 — Domain Services

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Explains domain services as cohesive domain policies without a natural entity owner, entity behavior, application-service boundaries, explicit time and external facts, testing, exercises, and review questions.

## Concepts already explained

Domain services express domain decisions in domain language and should remain independent of HTTP and persistence. They do not provide transaction atomicity; application and database boundaries enforce authorization and consistency.

## Terminology established

Domain service, domain policy, aggregate, domain port, application service, invariant.

## Examples used

Pricing policy, account transfer, currency conversion, and explicit external-rate ports.

## Cross-references

- [Chapter 200 — Aggregates](../../volumes/13-architecture/200-aggregates.md)
- [Chapter 204 — Application Services](../../volumes/13-architecture/204-application-services.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 203 — Domain Events: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XIII proofread. Domain-model guidance links to Fowler and PHP documentation.
