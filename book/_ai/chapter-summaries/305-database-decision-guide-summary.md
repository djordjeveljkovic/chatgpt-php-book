# AI Summary — Chapter 305 — Database Decision Guide

- Status: complete
- Volume: Volume 21 — REFERENCE
- Last updated: 2026-09-17

## Written material

Chapter 305 guides decisions among PHP, SQL, indexes, constraints, transactions, locks, replicas, caches, queues, and projections. It covers invariants, query plans, concurrency, pagination, tenancy, migration, reconciliation, a decision worksheet, exercises, and the Chapter 306 handoff.

## Concepts already explained

Invariant; authority; selectivity; query plan; constraint; transaction; isolation; deadlock; optimistic/pessimistic concurrency; read-after-write; keyset cursor; replica lag; projection; reconciliation.

## Terminology established

Reservation conflict, ORM query, tenant-scoped search, replica read, cache/projection, expand-and-contract migration, bounded export/backfill.

## Examples used

None.

## Cross-references

[Chapter 106 — Prepared Statements](../../volumes/08-databases/106-prepared-statements.md); [Chapter 110 — Query Plans](../../volumes/08-databases/110-query-plans.md); [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md); [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md); [Chapter 119 — Pagination](../../volumes/08-databases/119-pagination.md); [Chapter 123 — Database versus PHP Responsibilities](../../volumes/08-databases/123-database-vs-php-responsibilities.md); [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md); [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md).

## Open threads

Continue with Chapter 306 — Performance Checklist.

## Exact next section

Chapter 306 — Performance Checklist: the Why This Matters section.

## Technical verification notes

The source contains decision tables and prose with no executable PHP blocks. Local Markdown links resolved and `git diff --check` passed. The chapter received a local editorial check for invariants, SQL placement, plans, constraints, transactions, locks, replicas, projections, tenancy, migration, and reconciliation. Live database and deployment integrations were not run.

## Writing notes

Keep this summary short and update it after every writing session.
