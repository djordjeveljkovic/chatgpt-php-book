# AI Summary — Chapter 118 — Concurrency

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains PHP worker concurrency, lost updates, atomic guarded updates, unique constraints, idempotency keys, optimistic versions, pessimistic serialization, isolation choices, queue interleavings, concurrency tests, exercises, and review questions.

## Concepts already explained

- Separate PHP requests overlap through shared database state even when each process handles one request at a time.
- Constraints, atomic updates, version checks, locks, and queue semantics protect different invariants.
- Isolation level is not a universal mutex; serialization failures and duplicate delivery require replay-safe handling.

## Terminology established

Read-modify-write race, atomic guarded update, unique constraint, idempotency key, compare-and-swap, serialization failure, queue interleaving.

## Examples used

- Lost-balance race and atomic debit.
- Unique email/idempotency indexes, version-checked updates, and row-locked job claims.

## Cross-references

- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
- [Chapter 115 — Isolation](../../volumes/08-databases/115-isolation.md)
- [Chapter 116 — Locks](../../volumes/08-databases/116-locks.md)
- [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 119 — Pagination: the Why This Matters section.

## Technical verification notes

PHP snippets and local links are covered by the consolidated Volume VIII proofread. Isolation and locking claims are qualified by PostgreSQL/MySQL documentation.
