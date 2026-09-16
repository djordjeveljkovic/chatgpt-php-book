# AI Summary — Chapter 119 — Pagination

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains offset and keyset pagination, deterministic ordering, opaque signed cursors, index alignment, snapshot limits, page sizes, counts, API contracts, testing, exercises, and review questions.

## Concepts already explained

- Offset pagination is simple but can scan deeply and shift as rows change.
- Keyset pagination continues from the complete ordered key and needs matching predicates and indexes.
- Cursors are untrusted API input and should be validated, scoped, and authenticated.
- `LIMIT + 1` can derive `has_more` without an exact count; stable multi-request snapshots require a separate design.

## Terminology established

Offset pagination, keyset pagination, cursor, tie-breaker, total order, page-size cap, snapshot consistency, `has_more`.

## Examples used

- Offset and keyset SQL for an article feed.
- Typed PHP cursor decoding with signature validation.

## Cross-references

- [Chapter 108 — Indexes](../../volumes/08-databases/108-indexes.md)
- [Chapter 110 — Query Plans](../../volumes/08-databases/110-query-plans.md)
- [Chapter 118 — Concurrency](../../volumes/08-databases/118-concurrency.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 120 — Large Datasets: the Why This Matters section.

## Technical verification notes

PHP snippets and local links are covered by the consolidated Volume VIII proofread. Offset and keyset behavior is qualified by PostgreSQL/MySQL documentation and PDO parameter rules.
