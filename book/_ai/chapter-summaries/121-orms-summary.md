# AI Summary — Chapter 121 — ORMs

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains row/object mapping, identity maps, units of work, hydration choices, N+1 queries, lazy loading, transaction ownership, query visibility, optimistic concurrency, and long-running ORM contexts.

## Concepts already explained

- ORM convenience does not replace SQL, index, transaction, or concurrency reasoning.
- Entity graphs, scalar projections, and read models suit different query shapes.
- `flush()` is distinct from a business transaction unless the installed ORM explicitly defines otherwise.
- Managed entities must be cleared in long-running jobs; stale objects need versions or locks.

## Terminology established

ORM, mapper, identity map, persistence context, unit of work, hydration, N+1, lazy loading, projection, optimistic version.

## Examples used

- PHP backed enum and order aggregate.
- Unit-of-work flush, lazy relationship loop, and transaction service.

## Cross-references

- [Chapter 118 — Concurrency](../../volumes/08-databases/118-concurrency.md)
- [Chapter 120 — Large Datasets](../../volumes/08-databases/120-large-datasets.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 122 — Query Builders: the Why This Matters section.

## Technical verification notes

Doctrine APIs are labeled illustrative/version-specific; ORM behavior is grounded in Doctrine documentation and database locking concepts.
