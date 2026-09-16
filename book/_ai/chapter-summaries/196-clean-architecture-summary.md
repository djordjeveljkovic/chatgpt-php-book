# AI Summary — Chapter 196 — Clean Architecture

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Covers entities, use cases, interface adapters, frameworks and drivers, dependency direction, model mapping, transactions, security, architecture checks, and test levels.

## Concepts already explained

Clean Architecture points dependencies inward. Separate request, command, entity, persistence, and response models where their contracts differ; keep ports narrow and outer details replaceable.

## Terminology established

Clean Architecture, dependency rule, entity, use case, interface adapter, driver, port, ring theater, model mapping.

## Examples used

CreateInvoice command, InvoiceStore port, use case, package structure, and architecture testing.

## Cross-references

- [Chapter 195 — Hexagonal Architecture](../../volumes/13-architecture/195-hexagonal-architecture.md)
- [Chapter 194 — Modular Monolith](../../volumes/13-architecture/194-modular-monolith.md)

## Exact next section

Chapter 197 — Domain-Driven Design: the Why This Matters section.

## Technical verification notes

PHP examples linted; Clean Architecture, Hexagonal Architecture, and PHP namespace references are linked.
