# AI Summary — Chapter 177 — Coupling

- Status: complete
- Volume: Volume 12 — DESIGN AND PATTERNS
- Last updated: 2026-09-16

## Written material

Explains necessary versus accidental coupling, dependency direction, ports and adapters, composition roots, typed data boundaries, shared database coupling, temporal/failure coupling, events, and testing.

## Concepts already explained

Coupling should follow stable contracts and explicit ownership. Timeouts, retries, transactions, and event schemas are part of dependency contracts.

## Terminology established

Coupling, port, adapter, composition root, temporal coupling, failure coupling, service locator, event schema.

## Examples used

PaymentGateway port, checkout service, explicit construction, outbox/event decisions, and dependency-graph review.

## Cross-references

- [Chapter 159 — Unit Tests](../../volumes/11-testing/159-unit-tests.md)
- [Chapter 185 — Dependency Injection](../../volumes/12-design-and-patterns/185-dependency-injection.md)

## Exact next section

Chapter 178 — Cohesion: the Why This Matters section.

## Technical verification notes

PHP examples linted; PHP, PHP-FIG, and Martin Fowler references are linked.
