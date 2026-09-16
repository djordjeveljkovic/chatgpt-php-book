# AI Summary — Chapter 116 — Locks

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains database-owned locks, pessimistic row locking, PDO transaction scope, lock granularity and duration, optimistic version checks, waits and timeouts, observability, testing, exercises, and review questions.

## Concepts already explained

- Locks protect shared database resources for a transaction; PHP process-local state cannot coordinate workers.
- A locking read and dependent write must share a short transaction, and the locked rows must represent the real invariant.
- Optimistic version checks are an alternative for infrequent conflicts; guarded updates require affected-row checks.
- Lock timeouts require bounded, replay-safe handling and database-specific diagnostics.

## Terminology established

Pessimistic lock, optimistic locking, guarded update, lock scope, lock granularity, lock wait, lock timeout, authoritative row.

## Examples used

- `FOR UPDATE` slot consumption with PDO transaction and rollback.
- Version-checked document update and no-overlap invariant discussion.

## Cross-references

- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
- [Chapter 115 — Isolation](../../volumes/08-databases/115-isolation.md)
- [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 117 — Deadlocks: the Why This Matters section.

## Technical verification notes

PHP example linting and link validation are part of the consolidated Volume VIII proofread. Lock syntax is explicitly qualified as PostgreSQL/MySQL-sensitive and linked to their official documentation.
