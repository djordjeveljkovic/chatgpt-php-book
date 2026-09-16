# AI Summary — Chapter 108 — Indexes

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains indexes as maintained access paths, B-tree lookup and range cost, selectivity, uniqueness, foreign-key indexing, a parameterized PDO repository, function/wildcard pitfalls, edge cases, write/storage costs, index migrations, testing, exercises, and review questions.

## Concepts already explained

- A B-tree lookup is approximately `O(log N + k)` index work for `k` matches, while a table scan is approximately `O(N)` row visits; actual page/cache/fetch costs determine the plan.
- Indexes add storage, cache pressure, write amplification, maintenance, and DDL operational costs.
- Optimizers may ignore an index when many rows match, data is small, statistics are stale, or expressions/collations/casts prevent an efficient seek.
- Unique constraints enforce concurrent uniqueness; application check-then-insert logic has a race.
- Production-shaped data and EXPLAIN are required to validate an index proposal.

## Terminology established

B-tree, access path, selectivity, cardinality, range scan, covering index, residual filter, write amplification, page split, index bloat, partial/filtered index.

## Examples used

- Customer/status/date order index and parameterized PDO lookup.
- Support-ticket organization/state/date index.
- Normalized email and unique constraint.
- Function and leading-wildcard anti-patterns.

## Cross-references

- [Chapter 107 — Query Design](../../volumes/08-databases/107-query-design.md)
- [Chapter 109 — Composite Indexes](../../volumes/08-databases/109-composite-indexes.md)
- [Chapter 110 — Query Plans](../../volumes/08-databases/110-query-plans.md)
- [Chapter 111 — EXPLAIN](../../volumes/08-databases/111-explain.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
- [Chapter 118 — Concurrency](../../volumes/08-databases/118-concurrency.md)
- [Chapter 119 — Pagination](../../volumes/08-databases/119-pagination.md)

## Open threads

Engine-specific index types, online/concurrent DDL, and expression/partial indexes require verification against the deployed database version.

## Exact next section

Chapter 109 — Composite Indexes: Why This Matters.

## Technical verification notes

PHP examples use strict types, PDO prepared statements, typed properties, and valid heredoc syntax. SQL examples are intentionally labeled by surrounding discussion where syntax is engine-specific. References point to PostgreSQL, MySQL, and PHP primary documentation.

## Writing notes

Chapter is complete; preserve the distinction between an index candidate and the optimizer's chosen plan.
