# AI Summary — Chapter 111 — EXPLAIN

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains estimated versus runtime EXPLAIN, PostgreSQL and MySQL syntax, plan-tree reading, estimate/actual comparison, parameterized PHP plan diagnostics, safe fixed-query allowlists, comparing index candidates, execution side effects, plan regression testing, operational/security concerns, exercises, and review questions.

## Concepts already explained

- Plain `EXPLAIN` generally reports the planned procedure; runtime options execute the statement and expose actual rows, loops, timing, buffers, or equivalent data.
- EXPLAIN output is engine/version/schema/statistics/parameter specific; cost units are not milliseconds.
- Read leaves upward, distinguish index conditions from residual filters, multiply work by loops, and find the first meaningful cardinality divergence.
- PostgreSQL `FORMAT JSON`, `ANALYZE`, and `BUFFERS`, MySQL `FORMAT=JSON` and supported `EXPLAIN ANALYZE`, and SQLite `EXPLAIN QUERY PLAN` have distinct grammars.
- Plan tooling must not accept arbitrary SQL; use fixed query definitions, bound values, access control, redaction, timeouts, and safe environments.

## Terminology established

Estimated plan, runtime/actual plan, EXPLAIN ANALYZE, plan node, index condition, filter, loops, buffer reads, plan regression, query allowlist.

## Examples used

- PostgreSQL and MySQL EXPLAIN statements.
- Parameterized `TicketPlanReader` PHP class.
- Unsafe arbitrary-SQL endpoint and fixed-query replacement.
- Candidate ticket index comparison and plan regression testing.

## Cross-references

- [Chapter 108 — Indexes](../../volumes/08-databases/108-indexes.md)
- [Chapter 109 — Composite Indexes](../../volumes/08-databases/109-composite-indexes.md)
- [Chapter 110 — Query Plans](../../volumes/08-databases/110-query-plans.md)
- [Chapter 119 — Pagination](../../volumes/08-databases/119-pagination.md)

## Open threads

EXPLAIN options and output fields are vendor/version-specific. Runtime analysis of writes, triggers, volatile functions, locks, and replicas requires operational review before use.

## Exact next section

Chapter 112 — Joins: Why This Matters.

## Technical verification notes

PHP snippets use strict types, typed properties, bound parameters, and valid nowdoc syntax. SQL syntax is separated by engine where needed. References point to PostgreSQL, MySQL, SQLite, and PHP primary documentation.

## Writing notes

Chapter is complete; preserve the warning that runtime plan options execute work.
