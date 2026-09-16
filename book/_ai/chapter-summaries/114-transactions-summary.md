# AI Summary — Chapter 114 — Transactions

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains ACID, PDO transaction lifecycle, narrow transaction scope, constraints, atomic conditional updates, idempotency keys, external effects, outbox pattern, retry classification, failure handling, observability, tests, exercises, and review questions.

## Concepts already explained

Commit/rollback; autocommit; connection-scoped transactions; schema constraints; idempotent retries; outbox delivery; transient versus permanent failures.

## Terminology established

Unit of work, ambiguous outcome, idempotency key, outbox, bounded retry, atomic conditional update.

## Examples used

PDO order plus line-item transaction; inventory conditional update; idempotency table; outbox workflow.

## Cross-references

Chapter 105 PDO; Chapter 107 query design; Chapters 115–118 for isolation, locks, deadlocks, and concurrency.

## Open threads

Detailed visibility, lock, deadlock, and retry behavior continues in Chapters 115–118.

## Exact next section

Chapter complete; next chapter is 115 Isolation.

## Technical verification notes

PHP transaction example uses `Throwable`, `inTransaction()`, prepared statements, and rollback on error. Links use official PHP PDO and PostgreSQL documentation. Engine-specific DDL/transaction and SQLSTATE retry behavior is explicitly qualified.
