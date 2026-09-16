# AI Summary — Chapter 123 — Database vs PHP Responsibilities

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains choosing the database or PHP boundary, set operations and constraints, domain orchestration, transaction/outbox contracts, measurement, authorization, testing, exercises, and review questions.

## Concepts already explained

- Databases are strong at durable facts, set work, constraints, joins, and aggregates; PHP is strong at domain policy, orchestration, and external systems.
- The decision depends on data volume, selectivity, consistency, indexes, domain complexity, and boundary cost rather than ideology.

## Terminology established

Database/PHP boundary, set work, domain orchestration, transaction contract, outbox boundary, result shape.

## Examples used

- SQL filtering versus PHP filtering comparison.
- Reservation transaction with durable outbox record and post-commit notification.

## Cross-references

- [Chapter 120 — Large Datasets](../../volumes/08-databases/120-large-datasets.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
- [Chapter 122 — Query Builders](../../volumes/08-databases/122-query-builders.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 124 — HTTP: the Why This Matters section.

## Technical verification notes

PHP snippets and local links are covered by the consolidated Volume VIII/IX proofread. Database/PHP boundary claims link to PDO and official PostgreSQL/MySQL documentation.
