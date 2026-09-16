---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 117
title: Deadlocks
slug: deadlocks
status: complete
summary: ../../_ai/chapter-summaries/117-deadlocks-summary.md
---

# Chapter 117 — Deadlocks

## Why This Matters

A lock wait is not always a slow query. Two transactions can each hold a resource the other needs, so neither can make progress. The database calls this a deadlock and aborts one transaction to break the cycle. The losing PHP request sees an exception even though its SQL was valid in isolation.

Deadlocks are a normal consequence of concurrent locking, not evidence that a database is broken. The engineering response is to make cycles less likely, recognize the database's deadlock error, retry a safe unit of work, and observe recurring patterns.

[Chapter 116](116-locks.md) explains lock scope and ordering. This chapter focuses on the cycle, diagnosis, retry boundary, and design practices that keep recovery bounded.

## Mental Model

Represent transactions as a waits-for graph:

```text
transaction A holds row 1 ──► waits for row 2
transaction B holds row 2 ──► waits for row 1
```

The arrows form a cycle. A database detects the cycle and chooses a victim, usually rolling back that transaction so the other can continue. Which transaction is chosen and which error code is returned are engine-specific.

This differs from ordinary contention:

| Situation | Result |
| --- | --- |
| A waits for B, and B eventually commits | A can continue |
| A waits for B, B waits for C, and C commits | The chain can unwind |
| A waits for B and B waits for A | A cycle requires a victim |
| A waits forever because no timeout or detection exists | Operational outage risk |

## A Common Example

Consider transferring money between two accounts. One request locks account 10 and then 20; another locks 20 and then 10. Both transactions can reach the second lock while holding the first. The business operation is valid, but the acquisition order is inconsistent.

The simplest prevention is a canonical order:

```php
<?php

declare(strict_types=1);

function transfer(PDO $db, int $from, int $to, int $cents): void
{
    if ($from === $to || $cents < 1) {
        throw new InvalidArgumentException('Invalid transfer');
    }

    [$first, $second] = $from < $to ? [$from, $to] : [$to, $from];
    $db->beginTransaction();

    try {
        $lock = $db->prepare(
            'SELECT id, balance_cents
             FROM accounts
             WHERE id IN (:first, :second)
             ORDER BY id
             FOR UPDATE'
        );
        $lock->execute(['first' => $first, 'second' => $second]);

        // Validate balances and apply both updates inside this transaction.
        $debit = $db->prepare(
            'UPDATE accounts
             SET balance_cents = balance_cents - :cents
             WHERE id = :id AND balance_cents >= :cents'
        );
        $debit->execute(['cents' => $cents, 'id' => $from]);
        if ($debit->rowCount() !== 1) {
            throw new RuntimeException('Insufficient funds');
        }

        $credit = $db->prepare(
            'UPDATE accounts
             SET balance_cents = balance_cents + :cents
             WHERE id = :id'
        );
        $credit->execute(['cents' => $cents, 'id' => $to]);
        $db->commit();
    } catch (Throwable $error) {
        if ($db->inTransaction()) {
            $db->rollBack();
        }

        throw $error;
    }
}
```

The `ORDER BY` documents the intended order, but the application should also make sure every code path uses the same ordering and that the query plan actually locks the intended rows. A single multi-row locking statement is often easier to reason about than separate, conditionally ordered statements.

## Other Sources of Cycles

Inconsistent row order is common, but not the only cause. Cycles can involve:

- multiple tables updated in different orders;
- foreign-key checks and index records;
- gap or predicate locks under an engine's isolation rules;
- triggers that write additional tables;
- background jobs and HTTP requests using different transaction boundaries;
- long transactions that hold locks while waiting for application work.

Do not infer the exact cycle from a PHP stack trace alone. Capture the database error code and inspect the engine's deadlock report or activity views. PostgreSQL can log deadlock details with its server logging settings; InnoDB can include the latest detected deadlock in its engine status output. Use the deployment engine's documentation and logs rather than copying diagnostic queries between products.

## Retry the Transaction, Not One Statement

If the database aborts a transaction, its transaction state is no longer usable. The retry must begin a fresh transaction and rerun the complete business operation, including reads and validation. Retrying only the final `UPDATE` can apply a decision based on stale data.

The operation must be safe to repeat. A transfer should have an idempotency key or a durable operation record so a client retry cannot create two transfers. A reservation can use a unique request identifier. External side effects should be committed through an outbox or otherwise separated from the retried database transaction.

```php
<?php

declare(strict_types=1);

function retryDeadlock(callable $operation, callable $isDeadlock, int $maxAttempts = 3): mixed
{
    for ($attempt = 1; $attempt <= $maxAttempts; $attempt++) {
        try {
            return $operation();
        } catch (Throwable $error) {
            if (!$isDeadlock($error) || $attempt === $maxAttempts) {
                throw $error;
            }

            usleep(random_int(10_000, 50_000) * $attempt);
        }
    }

    throw new LogicException('Unreachable');
}
```

The detector must use the driver and database error code, not a substring search over a localized message. The backoff should be bounded and jittered. Retrying a deadlock indefinitely can turn a temporary conflict into a connection and queue outage.

## Observability and Diagnosis

Record the database operation name, attempt number, duration, and sanitized error code. Correlate the application request ID with the database log when the driver exposes a session identifier. Track deadlock count and retry success rate separately from ordinary query latency.

When a deadlock occurs, collect the engine's report: participating transactions, held locks, requested locks, SQL fingerprints, indexes, and transaction age. Reproduce with two controlled connections when possible. Look for a stable ordering violation, an unexpectedly broad predicate, or work that should have happened after commit.

A deadlock rate of zero in a small test is not proof that production cannot deadlock. Workload shape, indexes, isolation, and transaction timing change the graph. The useful target is bounded impact and a design whose recurring cycles can be eliminated.

## Common Mistakes

- Treating a deadlock as an application-fatal unrecoverable error every time.
- Retrying one statement after the transaction has been aborted.
- Retrying a non-idempotent operation without a durable deduplication key.
- Sleeping for a fixed long interval on every retry.
- Logging raw secrets or full parameter values while diagnosing SQL.
- Fixing one code path's lock order while another path uses the reverse order.
- Assuming a unique index removes all multi-row or multi-table deadlocks.

## Testing

Create an integration test with two connections and a deliberate barrier: connection A locks row 1, connection B locks row 2, then each attempts the other's row. Assert that one transaction receives the database-specific deadlock error and that the surviving transaction commits. Test the retry wrapper with a fake detector, including maximum attempts and a non-deadlock exception that must not be retried.

The test should be opt-in for engines whose deadlock scheduling is nondeterministic, but the production retry and lock-ordering policy should still be exercised against the real engine in a concurrency test environment.

## Senior Engineer Thinking

Deadlock handling has two layers: prevention and recovery. Canonical ordering, short transactions, and narrow predicates reduce cycles. A bounded, idempotent retry makes the remaining cycles survivable. Both are required; retry alone hides design problems, and prevention alone assumes an impossible level of control over every future query.

## Exercises

1. Draw the waits-for graph for two transfers that lock accounts in opposite orders, then change the implementation to a canonical order.
2. Implement a driver-specific deadlock classifier and document the error codes it accepts.
3. Add an idempotency record to a retried transfer and explain how it prevents duplicate external effects.
4. Design a two-connection integration test with a barrier and a bounded retry.

## Review Questions

1. What makes a lock wait a deadlock?
2. Why must a retry restart the whole transaction?
3. Which operations are unsafe to retry without an idempotency key?
4. Why can fixed backoff create synchronized retry storms?
5. What evidence should an engineer collect from the database after a deadlock?

## Summary

A deadlock is a cycle of transactions waiting for one another. Use consistent lock ordering and short transactions to reduce cycles, then retry a complete idempotent operation in a fresh transaction with bounded jittered backoff. Classify errors by database code, observe retry outcomes, and use engine diagnostics to remove recurring cycles.

## References

- [PostgreSQL: Deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html#EXPLICIT-LOCKING-DEADLOCKS)
- [PostgreSQL: Monitoring locks](https://www.postgresql.org/docs/current/monitoring-locks.html)
- [MySQL: InnoDB deadlocks](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks.html)
- [PHP Manual: PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)
