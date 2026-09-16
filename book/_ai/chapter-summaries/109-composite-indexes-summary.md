# AI Summary — Chapter 109 — Composite Indexes

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains tuple ordering and leftmost-prefix behavior, equality/range/order column selection, covering indexes, independent versus composite indexes, stable keyset pagination, engine-specific edge cases, index lifecycle and write cost, testing, exercises, and review questions.

## Concepts already explained

- `(a, b, c)` is ordered lexicographically; leading prefixes are generally more useful for navigation than predicates on later columns alone.
- Equality predicates commonly precede a range predicate; later columns may be residual filtering after a range begins.
- A covering index can avoid base-table lookups but increases storage and write work; PostgreSQL `INCLUDE` is vendor-specific.
- Pagination needs a unique tie-breaker and an index aligned with the ordering.
- Composite-index design must be tested with skew, realistic row counts, and the target engine/version.

## Terminology established

Composite index, leftmost prefix, lexicographic order, equality prefix, range boundary, covering/index-only scan, included column, keyset pagination, stable tie-breaker, redundant index.

## Examples used

- Events `(account_id, kind, occurred_at)` index with PDO.
- Competing `jobs` query shapes and candidate indexes.
- PostgreSQL `INCLUDE` covering index.
- Conversation message keyset pagination using `(sent_at, id)`.

## Cross-references

- [Chapter 108 — Indexes](../../volumes/08-databases/108-indexes.md)
- [Chapter 110 — Query Plans](../../volumes/08-databases/110-query-plans.md)
- [Chapter 111 — EXPLAIN](../../volumes/08-databases/111-explain.md)
- [Chapter 119 — Pagination](../../volumes/08-databases/119-pagination.md)

## Open threads

Skip scans, index intersection, row-value comparison, mixed directions, included columns, and partial indexes are engine/version-specific and should be checked before production use.

## Exact next section

Chapter 110 — Query Plans: Why This Matters.

## Technical verification notes

PHP examples use valid strict-typed PDO and keyset-boundary code. PostgreSQL-specific `INCLUDE` is labeled as such; generic composite-index claims are qualified by engine behavior. References point to PostgreSQL, MySQL, and PHP primary documentation.

## Writing notes

Chapter is complete; retain the distinction between filtering eventually and seeking a narrow ordered range.
