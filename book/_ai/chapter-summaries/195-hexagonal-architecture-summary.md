# AI Summary — Chapter 195 — Hexagonal Architecture

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Explains primary and secondary ports, adapters, composition roots, narrow capabilities, contract tests, transactions, outbox, provider idempotency, and failure translation.

## Concepts already explained

Hexagonal architecture keeps core policy independent from external technology. Ports clarify dependency direction, while adapters own protocol translation and infrastructure failures.

## Terminology established

Hexagonal architecture, Ports and Adapters, driving port, driven port, adapter, composition root, contract test.

## Examples used

PlaceOrder driving and driven ports, service implementation, adapter responsibilities, and outbox recovery.

## Cross-references

- [Chapter 193 — Layered Architecture](../../volumes/13-architecture/193-layered-architecture.md)
- [Chapter 196 — Clean Architecture](../../volumes/13-architecture/196-clean-architecture.md)

## Exact next section

Chapter 196 — Clean Architecture: the Why This Matters section.

## Technical verification notes

PHP examples linted; Alistair Cockburn, PHP-FIG, and PHP references are linked.
