# AI Summary — Chapter 198 — Entities

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Entities represent identity-bearing domain concepts with lifecycle and behavior. The chapter covers identity scope, equality, invariant-preserving methods, creation and reconstitution, persistence mapping, aggregate ownership, optimistic versions, and testing.

## Concepts already explained

Entity identity, equality, lifecycle, creation, reconstitution, invariant, aggregate root, optimistic version, and explicit mapping.

## Examples used

PHP ReservationId and Reservation identity, Account transitions, Subscription reconstitution, and versioned documents.

## Cross-references

- [Chapter 197 — Domain-Driven Design](../../volumes/13-architecture/197-domain-driven-design.md)
- [Chapter 199 — Value Objects](../../volumes/13-architecture/199-value-objects.md)
- [Chapter 200 — Aggregates](../../volumes/13-architecture/200-aggregates.md)

## Open threads

Continue with Chapter 199 on immutable value-based concepts.

## Exact next section

Chapter 199 — Value Objects: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; persistence and concurrency integration tests were not run.
