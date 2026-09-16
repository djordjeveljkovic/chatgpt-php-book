# AI Summary — Chapter 208 — When Not to Use Microservices

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

The chapter presents microservices as a trade-off involving independent scaling, ownership, deployment, technology, and fault isolation versus distributed-system and operational cost. It covers modular monoliths, consistency and shared data, team readiness, extraction signals, staged migration, alternatives, testing, and operations.

## Concepts already explained

Modular monolith, service boundary, independent deployment, data ownership, distributed transaction, contract, extraction signal, adapter, operational readiness, and reconciliation.

## Examples used

A PHP in-process BillingPort, a modular checkout service, and a staged extraction path to a remote billing implementation.

## Cross-references

- [Chapter 194 — Modular Monolith](../../volumes/13-architecture/194-modular-monolith.md)
- [Chapter 195 — Hexagonal Architecture](../../volumes/13-architecture/195-hexagonal-architecture.md)
- [Chapter 206 — Distributed Systems](../../volumes/13-architecture/206-distributed-systems.md)
- [Chapter 207 — Microservices](../../volumes/13-architecture/207-microservices.md)

## Open threads

Continue with Volume XIV, Chapter 209 on what frameworks actually do.

## Exact next section

Chapter 209 — What Frameworks Actually Do: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; service extraction and deployment integration tests were not run.
