# AI Summary — Chapter 180 — Immutability

- Status: complete
- Volume: Volume 12 — DESIGN AND PATTERNS
- Last updated: 2026-09-16

## Written material

Covers value objects, readonly properties/classes, normalization, persistent updates, DateTimeImmutable, shallow versus deep immutability, serialization, caching, concurrency, and testing.

## Concepts already explained

Readonly prevents reassignment but does not make nested objects deeply immutable. Local immutable values help with aliasing and retries, while database state still needs transactions, versions, and idempotency.

## Terminology established

Immutability, readonly, value object, persistent update, aliasing, deep immutability, immutable snapshot, reconstitution.

## Examples used

EmailAddress normalization, RetryPolicy persistent update, DateTimeImmutable, immutable event DTOs, and ownership boundaries.

## Cross-references

- [Chapter 179 — Encapsulation](../../volumes/12-design-and-patterns/179-encapsulation.md)
- [Chapter 141 — Idempotency](../../volumes/09-http-and-application-development/141-idempotency.md)

## Exact next section

Chapter 181 — SOLID: the Why This Matters section.

## Technical verification notes

PHP examples linted; readonly, DateTimeImmutable, and PHP runtime references are linked.
