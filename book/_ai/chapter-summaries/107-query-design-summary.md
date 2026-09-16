# AI Summary — Chapter 107 — Query Design

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains result grain, predicates and nulls, projections, `EXISTS`, deterministic ordering, N+1 queries, query shape and indexes, repository boundaries, failure behavior, testing, exercises, and review questions.

## Concepts already explained

- Query grain states what one output row represents and exposes fan-out errors.
- SQL should express filtering, existence, ordering, and aggregation close to the data when the database is the right boundary.
- Query count alone is not a performance metric; compare rows, bytes, scans, memory, and plans.
- Repositories should validate limits, map results, and define retries and failure outcomes.

## Terminology established

Query grain, cardinality, projection, three-valued logic, deterministic ordering, N+1 query, query fingerprint, plan stability.

## Examples used

- `EXISTS` reservation conflict check.
- Half-open time predicate and bounded article repository query.

## Cross-references

- [Chapter 104 — SQL for PHP Developers](../../volumes/08-databases/104-sql-for-php-developers.md)
- [Chapter 108 — Indexes](../../volumes/08-databases/108-indexes.md)
- [Chapter 112 — Joins](../../volumes/08-databases/112-joins.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 108 — Indexes: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated Volume VIII proofread. Query behavior is qualified with PostgreSQL/MySQL references and PDO documentation.
