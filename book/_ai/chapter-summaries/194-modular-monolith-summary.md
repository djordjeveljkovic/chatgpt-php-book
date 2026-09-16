# AI Summary — Chapter 194 — Modular Monolith

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Covers module ownership, public commands and events, dependency enforcement, shared database concerns, synchronous and asynchronous collaboration, migrations, authorization, observability, and testing.

## Concepts already explained

A modular monolith is one deployment with explicit business boundaries, owned state, and narrow APIs. Outbox, idempotency, schema versioning, and architecture checks support reliable module collaboration.

## Terminology established

Modular monolith, module API, module ownership, internal reach-through, projection, outbox, event ID, architecture check.

## Examples used

Orders module API, Billing/Inventory boundary design, and idempotent event consumer.

## Cross-references

- [Chapter 189 — Behavioral Patterns](../../volumes/12-design-and-patterns/189-behavioral-patterns.md)
- [Chapter 195 — Hexagonal Architecture](../../volumes/13-architecture/195-hexagonal-architecture.md)

## Exact next section

Chapter 195 — Hexagonal Architecture: the Why This Matters section.

## Technical verification notes

PHP examples linted; Martin Fowler and PHP namespace references are linked.
