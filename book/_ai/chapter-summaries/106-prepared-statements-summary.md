# AI Summary — Chapter 106 — Prepared Statements

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains SQL/data separation, PDO placeholders and binding, repeated parameters, dynamic `IN` lists and identifiers, allow-listed ordering, transactions, plan reuse, testing, security limits, exercises, and review questions.

## Concepts already explained

- Placeholders represent values, not table names, columns, keywords, or arbitrary SQL structure.
- Explicit binding communicates driver types but does not validate domain input.
- Dynamic lists require one trusted placeholder per value and a bounded empty-list policy.
- Prepared statements prevent value injection but do not provide authorization, constraints, transaction correctness, or efficient plans.

## Terminology established

Prepared statement, placeholder, value boundary, dynamic SQL structure, allow-list, emulated prepare, guarded parameter, driver type.

## Examples used

- PDO named parameters and explicit `bindValue()` types.
- Dynamic `IN` placeholders and allow-listed sort fragments.

## Cross-references

- [Chapter 105 — PDO](../../volumes/08-databases/105-pdo.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
- [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 107 — Query Design: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated Volume VIII proofread. PDO behavior is qualified against the PHP Manual and driver-specific documentation.
