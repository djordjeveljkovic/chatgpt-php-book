# AI Summary — Chapter 201 — Repositories

- Status: complete
- Volume: Volume 13 — ARCHITECTURE
- Last updated: 2026-09-16

## Written material

Repositories are domain-oriented persistence contracts. The chapter covers aggregate boundaries, missing and locking semantics, PDO mapping, transactions, optimistic versions, pagination, read models, caching, identity maps, and shared adapter tests.

## Concepts already explained

Repository port, aggregate repository, read-model repository, row lock, optimistic version, tenant scope, identity map, transaction boundary, and adapter contract test.

## Examples used

An OrderRepository interface, a PDO adapter with tenant scoping and locking, and an application-level confirmation transaction.

## Cross-references

- [Chapter 197 — Domain-Driven Design](../../volumes/13-architecture/197-domain-driven-design.md)
- [Chapter 200 — Aggregates](../../volumes/13-architecture/200-aggregates.md)
- [Chapter 190 — Enterprise Patterns](../../volumes/12-design-and-patterns/190-enterprise-patterns.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)

## Open threads

Continue with Chapter 202 on domain services.

## Exact next section

Chapter 202 — Domain Services: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; live database locking and pagination tests were not run.
