---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 118
title: Concurrency
slug: concurrency
status: complete
summary: ../../_ai/chapter-summaries/118-concurrency-summary.md
---

# Chapter 118 — Concurrency

## Why This Matters

An application can have concurrent requests even when each PHP process executes one request at a time. Multiple PHP-FPM workers, queue consumers, scheduled commands, and operators can all access the same database. If a decision is made from a value that another worker can change, the decision needs a database-level guarantee.

Concurrency bugs often pass ordinary tests because the failing interleaving is small and timing-dependent. The fix is to name the invariant, identify the read-modify-write window, and make the database perform the check and update atomically or under an appropriate lock.

Chapters 114–117 cover transactions, isolation, locks, and deadlocks. This chapter compares common concurrency-control patterns and their failure modes.

## The Read-Modify-Write Race

This code is unsafe for a shared counter:

```php
$balance = (int) $db->query('SELECT balance_cents FROM accounts WHERE id = 7')->fetchColumn();
$balance -= 100;
$db->prepare('UPDATE accounts SET balance_cents = :balance WHERE id = 7')
    ->execute(['balance' => $balance]);
```

Two requests can read 1,000, both compute 900, and both write 900. One debit disappears. A transaction alone does not necessarily fix this: under common isolation levels, both transactions may read the same snapshot unless the read locks the row or the write detects a version change.

Prefer an atomic guarded update when the rule fits one statement:

```sql
UPDATE accounts
SET balance_cents = balance_cents - :amount
WHERE id = :id AND balance_cents >= :amount;
```

The application checks that one row changed. The database serializes competing updates to the same row and evaluates the predicate against the current row version. This is often shorter and more scalable than a separate read followed by a write.

## Unique Constraints as Concurrency Control

Application code can check whether an email or idempotency key exists, but two requests can both observe absence before either inserts. A unique constraint makes the invariant authoritative:

```sql
CREATE UNIQUE INDEX user_email_normalized_unique
    ON users (email_normalized);
```

Insert and handle the duplicate-key error. Do not first issue a `SELECT` and treat it as a guarantee. The pre-check can improve a user-facing message, but the constraint is the final arbiter.

Likewise, a reservation request can carry a unique client-generated key:

```sql
CREATE UNIQUE INDEX reservation_request_unique
    ON reservations (request_id);
```

On retry, the application loads the existing result by `request_id` and returns it rather than creating a second reservation. The key should be scoped to the operation and retained long enough to cover client and queue retries.

## Optimistic Version Checks

For an aggregate that may be edited by different users, store a version and update conditionally:

```sql
UPDATE projects
SET name = :name,
    version = version + 1
WHERE id = :id AND version = :version;
```

If the affected-row count is zero, report a conflict or reload and merge. Do not silently overwrite the other writer. This pattern is a database form of compare-and-swap. It works well when collisions are uncommon and the caller can resolve them.

## Pessimistic Serialization

When a conflict is likely or the decision spans multiple rows, lock the authoritative rows in a transaction:

```sql
BEGIN;

SELECT id, state
FROM jobs
WHERE id = :id
FOR UPDATE;

UPDATE jobs
SET state = 'claimed', worker_id = :worker_id
WHERE id = :id AND state = 'ready';

COMMIT;
```

Only one worker can claim the same row. Queue consumers may use engine-specific `SKIP LOCKED` behavior to avoid waiting on already claimed work, but this changes which rows are observed and must be part of the queue semantics. [Chapter 116](116-locks.md) covers scope and timeouts.

## Isolation Is a Choice, Not a Mutex

Isolation controls which concurrent effects a transaction can observe; it does not automatically express every business invariant. `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE` have different costs and behavior across engines. A serializable transaction may abort with a serialization failure and require the same kind of bounded retry as a deadlock.

Choose the weakest level that supports the invariant when it produces simpler and more predictable operations, then add explicit locks or constraints where needed. Raising isolation globally because one query is unsafe can increase contention and abort rates for unrelated work.

## PHP Process and Queue Concurrency

Never assume a static property, singleton, or in-memory array is shared by PHP-FPM workers. It can coordinate calls inside one process but cannot protect database state across workers or hosts. A long-running worker also needs to release or recreate database connections after failures according to the framework and driver policy.

Queues add another interleaving: a message may be delivered twice, out of order, or after a timeout while the first worker is still running. Make the handler idempotent, use a durable status or idempotency record, and ensure the transaction commits the state transition atomically. A visibility timeout is not a lock on arbitrary database rows.

## Concurrency Test Design

Concurrency tests need control over interleavings. Use two real database connections and a barrier or explicit pause after the first read/lock. Test the invariant rather than a particular timing. Useful scenarios include two debits, two unique inserts, two job claims, a version conflict, a deadlock retry, and a transaction that rolls back after a process-side exception.

Run these tests against the production database family where lock and isolation semantics matter. SQLite is useful for repository tests but does not reproduce every server's row-locking and concurrency behavior.

## Common Mistakes

- Believing separate PHP requests cannot overlap.
- Reading a value, making a decision in PHP, and writing it back without a guard.
- Treating an existence pre-check as a uniqueness guarantee.
- Overwriting a version conflict instead of reporting it.
- Using a process-local cache as the source of truth for mutable shared state.
- Retrying serialization failures or deadlocks without replay-safe operations.
- Adding `SKIP LOCKED` without defining fairness and starvation behavior.
- Testing only with one connection or an in-memory database.

## Senior Engineer Thinking

Concurrency control is a data-model decision. Constraints protect facts that must always be true; atomic updates protect single-row transitions; optimistic versions protect edits with conflict reporting; locks serialize multi-row decisions; queues shape when work may occur. Choose the smallest mechanism that proves the invariant and make failure visible to callers and operators.

## Exercises

1. Rewrite a read-modify-write counter as an atomic guarded update and define its affected-row outcomes.
2. Add an idempotency key and unique constraint to a create operation. Describe the response for a first request, a retry, and a conflicting payload using the same key.
3. Design a two-connection test for a lost update and a version conflict.
4. Compare `FOR UPDATE`, optimistic versions, and `SERIALIZABLE` for a no-overlap reservation operation.

## Review Questions

1. Why does a transaction alone not necessarily prevent a lost update?
2. Which layer should enforce uniqueness when requests can race?
3. When is an atomic SQL update preferable to a locked read?
4. Why is a queue delivery timeout not a database lock?
5. What must be true before a transaction can safely be retried?

## Summary

Concurrent PHP workers share database state even though each process handles one request at a time. Protect read-modify-write operations with atomic predicates, constraints, optimistic versions, or locks chosen for the invariant. Make retries and queue handlers idempotent, and test interleavings with real concurrent connections and the production database family.

## References

- [PostgreSQL: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [MySQL: InnoDB transaction isolation](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [PHP Manual: PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)
