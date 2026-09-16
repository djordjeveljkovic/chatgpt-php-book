# AI Summary — Chapter 193 — Layered Architecture

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Explains presentation, application, domain, and infrastructure layers; dependency direction; mapping; transaction ownership; failure and security boundaries; architecture testing; and trade-offs with vertical slices.

## Concepts already explained

Layers are meaningful only when dependency direction and contracts are enforced. Keep framework and persistence details at the edges and map errors to stable outcomes.

## Terminology established

Presentation layer, application layer, domain layer, infrastructure layer, dependency direction, boundary leakage, architecture test.

## Examples used

RegisterAccount command and handler, request flow, transaction policy, and error mapping.

## Cross-references

- [Chapter 192 — Simple Architecture](../../volumes/13-architecture/192-simple-architecture.md)
- [Chapter 195 — Hexagonal Architecture](../../volumes/13-architecture/195-hexagonal-architecture.md)

## Exact next section

Chapter 194 — Modular Monolith: the Why This Matters section.

## Technical verification notes

PHP examples linted; PHP-FIG and Martin Fowler references are linked.
