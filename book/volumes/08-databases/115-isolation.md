---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 115
title: Isolation
slug: isolation
status: complete
summary: ../../_ai/chapter-summaries/115-isolation-summary.md
---

# Chapter 115 — Isolation

## Why This Matters

Two PHP workers can run the same code at the same time. A transaction keeps each worker's changes coherent, but isolation determines which concurrent changes each one can see. If an application reads a balance, makes a decision, and writes a new balance, the result depends on whether another transaction can change the relevant rows between those steps.

Isolation is a correctness setting, not merely a performance knob. Choose it from the invariant and the acceptable anomalies, then test the behavior on the actual database engine and version.

## The Main Anomalies

A dirty read observes data another transaction has written but not committed. If that transaction rolls back, the reader acted on a value that never existed as committed state. Standard `READ COMMITTED` prevents dirty reads.

A non-repeatable read occurs when a transaction reads a row twice and another transaction commits an update between the reads. A phantom is a changed set of rows: the second execution of a predicate sees a newly inserted or deleted matching row. A lost update occurs when two workers read the same old value and later overwrite each other's result without coordination. Write skew occurs when each transaction updates a different row after reading a shared rule, allowing the combined state to violate that rule.

The names describe observable behavior, not one universal implementation. MVCC, locking, and predicate protection vary by engine.

## Isolation Levels

The SQL standard names `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE`, but implementations differ in exact guarantees. PostgreSQL documents `READ COMMITTED` as its default and uses a statement-level snapshot there; its `REPEATABLE READ` uses a transaction-level snapshot and can abort some concurrent anomalies. PostgreSQL's `READ UNCOMMITTED` behaves as `READ COMMITTED`. MySQL/InnoDB has different defaults and semantics, so do not copy a PostgreSQL assumption into a MySQL application.

`READ UNCOMMITTED` permits the weakest observations and is rarely suitable for business decisions. `READ COMMITTED` gives each statement a current committed view and is often a good default for independent reads. `REPEATABLE READ` gives a stable view in engines that provide transaction snapshots, but does not automatically protect every business predicate. `SERIALIZABLE` provides the strongest target behavior by making conflicting schedules appear serial, often by blocking or aborting a transaction; applications must retry serialization failures.

Choose the lowest level that protects the invariant, and add explicit row or predicate coordination where the engine requires it. Higher isolation can reduce concurrency through waits, memory, or retries.

## A Concurrent Invariant

Suppose a room may have at most one active reservation. Two workers each run:

```sql
SELECT COUNT(*)
FROM reservations
WHERE room_id = :room_id
  AND starts_at < :ends_at
  AND ends_at > :starts_at;
```

Both can observe zero under a weak coordination strategy and both insert. A count check alone is not a guarantee. Prefer a database constraint that represents the rule when the engine supports it, or use an appropriate lock/serializable design. In PostgreSQL, an exclusion constraint can encode non-overlapping ranges; in other engines, a unique key or carefully designed locking protocol may be needed.

For a simple counter, make the update itself conditional:

```sql
UPDATE inventory
SET available = available - :quantity
WHERE product_id = :product_id
  AND available >= :quantity;
```

Run it in a transaction and check the affected-row count. This avoids the read-modify-write gap for that invariant. The exact behavior under contention still depends on the engine and isolation level.

## Setting Isolation with PDO

Set the level using database-specific SQL at the correct transaction boundary. PostgreSQL supports `SET TRANSACTION` for the current transaction and requires it before the first data statement:

```php
<?php
declare(strict_types=1);

$pdo->beginTransaction();
try {
    $pdo->exec('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');

    $stmt = $pdo->prepare(
        'SELECT available FROM inventory WHERE product_id = :product_id'
    );
    $stmt->execute(['product_id' => $productId]);
    $available = (int) $stmt->fetchColumn();

    if ($available < $quantity) {
        throw new RuntimeException('Insufficient inventory');
    }

    $update = $pdo->prepare(
        'UPDATE inventory SET available = available - :quantity
         WHERE product_id = :product_id'
    );
    $update->execute([
        'quantity' => $quantity,
        'product_id' => $productId,
    ]);
    $pdo->commit();
} catch (Throwable $e) {
    if ($pdo->inTransaction()) {
        $pdo->rollBack();
    }
    throw $e;
}
```

A production retry wrapper should catch only the engine's documented serialization/deadlock errors, create a fresh transaction, and repeat the complete operation. Never retry by calling `beginTransaction()` while a failed transaction is still active.

## Snapshot and Connection Pool Concerns

A snapshot belongs to a transaction or statement according to the engine. A pooled or persistent connection can retain session settings, temporary objects, or an aborted transaction if the application does not reset it. After an exception, roll back before returning a connection to the pool; set important session options explicitly rather than trusting a previous request's state.

Read-only reports may tolerate a slightly changing view. Financial decisions, inventory reservations, and uniqueness rules need a documented consistency guarantee. Avoid holding a snapshot open while streaming a slow response, because long-lived snapshots can delay cleanup of old row versions in MVCC systems.

## Performance and Failure

Stronger isolation can increase lock waits, transaction aborts, and retry volume. Monitor transaction age, wait duration, serialization failures, deadlocks, and statement latency. A timeout is not proof that the server did nothing: if the outcome is ambiguous, use an idempotency key or query durable state before repeating a command.

Do not fix every anomaly by selecting `SERIALIZABLE` globally. First encode the invariant with a constraint or atomic statement where possible, then use targeted locking or a stronger transaction for the remaining coordination. Keep retry logic bounded and expose exhaustion as an operational error.

## Testing Concurrency

A sequential unit test cannot prove isolation. Use two database connections and synchronization barriers so test transaction A pauses after its read while transaction B updates or inserts. Assert the result and the expected error or wait behavior for the selected engine. Run these tests against the same database family used in production; SQLite's locking and isolation model is not a substitute for PostgreSQL or MySQL behavior.

## Senior Engineer Thinking

Name the invariant first: “no two active reservations overlap,” “stock never becomes negative,” or “a report can use a statement-consistent view.” Map the invariant to a constraint, atomic statement, lock, or isolation level. Document which failures are expected and retryable. Isolation is a contract between database behavior and application recovery; neither side is sufficient alone.

## Exercises

1. Build a two-connection test that demonstrates a non-repeatable read under `READ COMMITTED` on your engine.
2. Rewrite the inventory example as one conditional `UPDATE`, then explain which race it removes.
3. Choose an isolation strategy for a reservation system and list the exact constraint, retry condition, and monitoring signals it requires.

## Review Questions

- What is the difference between a non-repeatable read and a phantom?
- Why does `SERIALIZABLE` often require retries?
- Why is a count-then-insert check insufficient for a uniqueness-like business rule?
- What must happen to a connection after a transaction error before it is reused?

## Summary

Isolation controls concurrent visibility and the anomalies a transaction can experience. Know the engine's actual guarantees, express invariants with constraints or atomic statements, use stronger isolation selectively, and retry complete transactions after documented serialization failures. Concurrency tests must use real database connections and synchronization rather than sequential mocks.

## References

- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [MySQL: InnoDB Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [PHP manual: PDO::setAttribute](https://www.php.net/manual/en/pdo.setattribute.php)
