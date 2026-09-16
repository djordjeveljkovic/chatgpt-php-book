# AI Summary — Chapter 207 — Microservices

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Covers service boundaries and ownership, synchronous APIs, events, data authority, sagas, outbox publishing, idempotency, timeouts, retries, circuit breakers, deployment, observability, security, migration, and testing.

## Concepts already explained

Microservices trade independent deployment, scaling, and failure isolation for network latency, partial failure, distributed consistency, operational overhead, and a larger security surface. Services should own the data that enforces their invariants.

## Terminology established

Microservice, service boundary, service ownership, synchronous API, event contract, saga, compensation, outbox, idempotency, circuit breaker, bulkhead, correlation ID, anti-corruption adapter.

## Examples used

InventoryClient port, OrderWorkflow, reservation idempotency, order saga, event versioning, and monolith extraction.

## Cross-references

- [Chapter 194 — Modular Monolith](../../volumes/13-architecture/194-modular-monolith.md)
- [Chapter 206 — Distributed Systems](../../volumes/13-architecture/206-distributed-systems.md)
- [Chapter 208 — When Not to Use Microservices](../../volumes/13-architecture/208-when-not-to-use-microservices.md)

## Exact next section

Chapter 208 — When Not to Use Microservices: the Why This Matters section.

## Technical verification notes

PHP example linted; HTTP, Martin Fowler, and OWASP references are linked.
