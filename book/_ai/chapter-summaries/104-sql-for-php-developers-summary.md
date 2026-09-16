# AI Summary — Chapter 104 — SQL for PHP Developers

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Introduces relational tables, rows and keys, SQL's declarative model, filtering, ordering, joins, aggregation, nulls, transactions, query cost, database/PHP boundaries, exercises, and review questions.

## Concepts already explained

- SQL expresses a result and lets the database choose an execution plan; query cost includes scans, joins, sorting, transfer, and PHP hydration.
- Primary/foreign keys and constraints represent invariants; filtering, ordering, aggregation, and pagination belong near the data when appropriate.
- SQL `NULL`, three-valued logic, half-open time ranges, and explicit ordering require deliberate handling.

## Terminology established

Relation, row, column, primary key, foreign key, constraint, predicate, projection, cardinality, `NULL`, query plan, round trip.

## Examples used

- Customers/orders schema and progressively improved queries.
- PHP/PDO query boundary, indexed filtering, aggregation, and transaction examples.

## Cross-references

- [Chapter 105 — PDO](../../volumes/08-databases/105-pdo.md)
- [Chapter 107 — Query Design](../../volumes/08-databases/107-query-design.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 105 — PDO: the Why This Matters section.

## Technical verification notes

Chapter examples and links are included in the consolidated Volume VIII proofread. Vendor-specific SQL differences are identified and linked to official PostgreSQL/MySQL documentation.
