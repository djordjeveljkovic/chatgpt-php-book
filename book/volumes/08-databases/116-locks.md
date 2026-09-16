---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 116
title: Locks
slug: locks
status: complete
summary: ../../_ai/chapter-summaries/116-locks-summary.md
---

# Chapter 116 — Locks

## Why This Matters

Two PHP requests can read the same database row and then try to change it. PHP-FPM workers are separate processes, so a local variable or in-process mutex cannot coordinate those requests. The database is the shared authority, and its locking rules determine whether the invariant survives concurrent work.

A lock is not a general “make this safe” switch. It protects a defined resource for a defined interval under a defined transaction. A lock held too broadly reduces throughput; a lock held too narrowly allows races. Reliable code states what is protected, acquires locks in a predictable order, and handles waiting, timeout, rollback, and retry.

Chapters 114 and 115 introduce transactions and isolation. This chapter concentrates on explicit row and table locks and on the PHP transaction boundary. [Chapter 117](117-deadlocks.md) explains what happens when lock acquisition forms a cycle.

## Mental Model

Think of a lock as a database-owned reservation on a resource:

```text
begin transaction
    ↓
acquire lock on rows or other resource
    ↓
read and validate current state
    ↓
write the invariant-preserving change
    ↓
commit (release) or rollback (release)
```

The lock is visible to other database sessions, not to ordinary PHP code. A transaction normally holds a row lock until commit or rollback, although exact lock types and release rules vary by database engine. A request that crashes does not leave a permanent lock: the database detects the broken session and rolls back its open transaction, subject to engine behavior and timeout detection.

## Pessimistic Row Locking

Pessimistic locking assumes a conflict is possible and locks the rows before making a decision. A reservation service can lock a court's current availability record, verify it, and update it in one transaction:

```sql
BEGIN;

SELECT available_slots
FROM court_days
WHERE court_id = :court_id AND day = :day
FOR UPDATE;

UPDATE court_days
SET available_slots = available_slots - 1
WHERE court_id = :court_id
  AND day = :day
  AND available_slots > 0;

COMMIT;
```

`FOR UPDATE` is common PostgreSQL and MySQL syntax for locking selected rows, but details such as which index records or gaps are locked depend on the engine, isolation level, and query plan. A lock does not make a query correct if the `WHERE` clause selects the wrong rows. The application should check the update count and roll back when no eligible row remains.

In PDO, the transaction must surround both the locking read and the write:

```php
<?php

declare(strict_types=1);

function consumeSlot(PDO $db, int $courtId, string $day): void
{
    $db->beginTransaction();

    try {
        $select = $db->prepare(
            'SELECT available_slots
             FROM court_days
             WHERE court_id = :court_id AND day = :day
             FOR UPDATE'
        );
        $select->execute(['court_id' => $courtId, 'day' => $day]);
        $slots = $select->fetchColumn();

        if ($slots === false || (int) $slots < 1) {
            throw new RuntimeException('No slot is available');
        }

        $update = $db->prepare(
            'UPDATE court_days
             SET available_slots = available_slots - 1
             WHERE court_id = :court_id AND day = :day'
        );
        $update->execute(['court_id' => $courtId, 'day' => $day]);
        $db->commit();
    } catch (Throwable $error) {
        if ($db->inTransaction()) {
            $db->rollBack();
        }

        throw $error;
    }
}
```

The transaction callback must not perform slow network calls while holding the lock. If payment or an external notification is required, record durable local state in the transaction and perform the external step after commit, usually through an outbox or queue.

## Lock Scope and Granularity

Locking one row usually permits more concurrency than locking a whole table, but the smaller scope is useful only when the invariant is represented by those rows. If a rule says “no overlapping reservations for a court,” locking one summary row is safe only if every writer uses that row. Otherwise two requests may lock different reservation rows and still create an overlap. The database may need a unique or exclusion constraint, a serializable transaction, or a deliberately locked schedule row.

Keep the transaction short. Do not load an entire report, render a template, or wait for an HTTP service while holding a lock. Select only the rows needed for the decision, update them, and commit. The lock duration is approximately the time from acquisition to transaction end, so slow application code directly increases wait time for other sessions.

The number and order of rows matter. If a transfer locks account A and then B, every transfer should lock the lower account ID first. A consistent order reduces deadlocks and makes waits easier to reason about. [Chapter 117](117-deadlocks.md) covers the cases that remain.

## Optimistic Locking

Pessimistic locks are not always the best fit. When conflicts are uncommon, an optimistic version column lets requests read without holding a lock and update only if the version is unchanged:

```sql
UPDATE documents
SET body = :body,
    version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE id = :id AND version = :expected_version;
```

The application checks that exactly one row changed. Zero rows means another writer won or the document disappeared; the caller can reload, merge, or report a conflict. This is not “no concurrency control.” The conditional update is the concurrency control, and the database evaluates it atomically.

Optimistic locking is useful for edit forms and APIs where a user may spend minutes editing a record. It is less useful for a hot counter with frequent collisions, where retries can overwhelm the database. Choose based on conflict frequency, acceptable retry cost, and the user-visible conflict policy.

## Lock Waits and Timeouts

A waiting transaction consumes a connection and keeps its own resources. Configure a bounded lock wait or statement timeout appropriate to the operation, and handle the database-specific timeout error. A timeout is a failure of this attempt, not proof that the operation should be retried immediately forever. Retry only an operation that is safe to repeat, with a small attempt limit and backoff.

Observe lock waits through the database's activity and lock views, slow-query logs, or monitoring integration. Log a request or operation ID, the transaction purpose, duration, and database error code. Do not log secrets or entire SQL parameter values by default. Query text and lock diagnostics differ by engine; consult PostgreSQL's [`pg_locks`](https://www.postgresql.org/docs/current/view-pg-locks.html) and MySQL's [InnoDB locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html) documentation for the deployment engine.

## Common Mistakes

- Starting a transaction after the read that made the decision.
- Assuming a PHP object lock coordinates other workers.
- Holding a database lock while calling an external service.
- Locking a row that does not represent the real invariant.
- Ignoring the affected-row count from a guarded update.
- Retrying every lock timeout without an idempotency key or attempt limit.
- Mixing lock syntax or semantics from one database engine into another.
- Reading a row with a lock and then performing an unguarded write through a different code path.

## Testing

Use at least two database connections to test lock behavior. Arrange for connection A to begin a transaction and acquire the lock, then have connection B attempt the competing operation with a short timeout. Assert that B waits or fails as the selected engine specifies, and that A's commit or rollback produces the expected result. Include tests for the guarded update returning zero rows and for rollback after an exception.

Unit tests of a PHP repository cannot prove the database's locking behavior. Keep a small integration suite against the same database family and version range used in production.

## Senior Engineer Thinking

Start with the invariant and its authoritative rows. Then decide whether a database lock, a conditional write, a constraint, or a combination provides the clearest guarantee. Measure contention before widening or removing locks. A lock is successful when it protects correctness at an acceptable cost, not when every query uses `FOR UPDATE`.

## Exercises

1. Implement a PDO repository method that decrements a stock row under a transaction and rejects a zero-row guarded update.
2. Write a two-connection integration test that demonstrates a lock wait and rollback.
3. Add a version column to an edit form and design the response when the version has changed.
4. Identify the rows that must be protected for a no-overlap reservation invariant and explain why locking one existing reservation is insufficient.

## Review Questions

1. What interval does a transaction lock normally cover?
2. Why must a locking read and its dependent write be in the same transaction?
3. When is optimistic locking a better fit than a pessimistic row lock?
4. Why can a lock be correct yet still cause an operational incident?
5. What should an application do after a lock timeout?

## Summary

Database locks coordinate concurrent sessions around shared state. Use a short transaction, lock rows that actually represent the invariant, check guarded write counts, and avoid external work while locks are held. Optimistic version checks are an alternative when conflicts are infrequent. Lock behavior is database-specific and must be tested with real concurrent connections.

## References

- [PHP Manual: PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL: `pg_locks`](https://www.postgresql.org/docs/current/view-pg-locks.html)
- [MySQL: InnoDB locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
