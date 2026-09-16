---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 114
title: Transactions
slug: transactions
status: complete
summary: ../../_ai/chapter-summaries/114-transactions-summary.md
---

# Chapter 114 — Transactions

## Why This Matters

A business operation often changes several rows. Placing an order may insert an order, insert its items, reserve stock, and record an event. If the process stops after only two changes, the database contains a state that never represented a valid business outcome. A transaction gives those database statements one commit point: readers see either the committed unit or none of it, subject to the database's isolation rules.

A transaction cannot make an external email, HTTP request, or filesystem write atomic with a database commit. Those boundaries need an explicit design such as an outbox and a retryable consumer.

## Mental Model and ACID

Atomicity makes the transaction all-or-nothing. Consistency means constraints and application invariants hold at commit; the database can enforce keys, foreign keys, checks, and unique rules, while application code must define wider business invariants. Isolation controls what concurrent transactions can observe. Durability means committed data survives the failure guarantees of the configured database and storage system.

A transaction is a database connection state, not a PHP object boundary. With PDO, `beginTransaction()` starts it, `commit()` makes changes durable according to the engine, and `rollBack()` abandons uncommitted changes. Autocommit means each statement is committed separately when no explicit transaction is active.

## A Minimal PDO Unit of Work

Use exception mode and a narrow `try` block. Keep validation that does not need the database outside the transaction, but perform all dependent writes inside it:

```php
<?php
declare(strict_types=1);

$pdo = new PDO($dsn, $username, $password, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
]);

$pdo->beginTransaction();
try {
    $insertOrder = $pdo->prepare(
        'INSERT INTO orders (id, customer_id, status, created_at)
         VALUES (:id, :customer_id, :status, :created_at)'
    );
    $insertOrder->execute([
        'id' => $orderId,
        'customer_id' => $customerId,
        'status' => 'pending',
        'created_at' => $createdAt,
    ]);

    $insertItem = $pdo->prepare(
        'INSERT INTO order_items (order_id, product_id, quantity, unit_price)
         VALUES (:order_id, :product_id, :quantity, :unit_price)'
    );
    foreach ($items as $item) {
        $insertItem->execute([
            'order_id' => $orderId,
            'product_id' => $item['product_id'],
            'quantity' => $item['quantity'],
            'unit_price' => $item['unit_price'],
        ]);
    }

    $pdo->commit();
} catch (Throwable $e) {
    if ($pdo->inTransaction()) {
        $pdo->rollBack();
    }
    throw $e;
}
```

The schema's foreign key, check constraints, and primary key remain the final defense. Application validation improves error messages but cannot replace database enforcement.

## Transaction Scope

A transaction should include the smallest set of statements that must succeed together. Do not hold a transaction open while waiting for a remote API, reading a large user upload, or rendering a response. Long transactions retain versions or locks, increase contention, and make failure recovery more expensive.

A transaction does not automatically include another connection. If a PHP service opens two database connections, each has independent transaction state. A failure between them can leave one committed and one rolled back. Use one connection for one local transaction, or adopt a distributed workflow deliberately; do not assume PDO coordinates it.

DDL and transaction behavior differs by engine. Some databases support transactional DDL for many operations; others implicitly commit particular schema statements. Consult the engine documentation before mixing migrations and data changes.

## Atomic State Changes and Idempotency

An application-level check followed by an update can race. Where possible, express the precondition in one statement and inspect affected rows:

```sql
UPDATE inventory
SET available = available - :quantity
WHERE product_id = :product_id
  AND available >= :quantity;
```

Inside a transaction, an affected-row count of one means the reservation succeeded; zero means the precondition was not met. The exact locking and isolation behavior is engine-specific, and the next chapters cover it in detail.

Retries create a second problem: the original request may have committed before the connection failed. Give externally retried commands an idempotency key with a unique constraint, and store the result associated with that key in the same transaction as the business change.

```sql
CREATE TABLE idempotency_keys (
    key         VARCHAR(100) PRIMARY KEY,
    result_code INTEGER NOT NULL,
    result_body  TEXT NOT NULL,
    created_at   TIMESTAMP NOT NULL
);
```

A duplicate key then becomes a deterministic lookup instead of a duplicate order.

## External Effects and the Outbox

Do not send an email between `beginTransaction()` and `commit()` and then assume a rollback retracts it. Conversely, committing and then sending an event can lose the event if PHP crashes in between. Insert an outbox row in the same transaction as the state change, commit, and let a worker publish the row with a claim/retry policy. The worker must tolerate duplicate delivery; consumers need an idempotency key or a deduplication table.

## Failure Modes and Retry Policy

Rollback on an exception, close or reset the connection, and preserve the original error context. A deadlock or serialization failure may be transient, but a unique-constraint violation or invalid foreign key usually is not. Retry only identified transient SQLSTATEs, with a small bounded attempt count and jitter. Re-run the complete transaction on a fresh snapshot; do not continue halfway through a failed one.

A process crash generally causes the database to recover uncommitted work, but durability depends on the engine's configuration and storage guarantees. Operational recovery is not a substitute for testing backup and restore.

## Testing and Observability

Integration tests should force an exception after each important statement and verify that no partial rows remain. Test duplicate idempotency keys and a retry after an ambiguous connection failure. Record transaction duration, rollback count, deadlock/serialization failures, and affected row counts without logging secrets or payment data.

## Senior Engineer Thinking

A transaction boundary should be stated in business terms: “create an order and its lines,” not “the controller method.” Identify the invariant, the database constraints that enforce it, the concurrent operations that can challenge it, and how a retry behaves. Then decide which effects belong in the transaction and which are delivered through an outbox or another durable workflow.

## Exercises

1. Add an idempotency-key flow to the order example and explain what happens when the client retries after a timeout.
2. Write an integration test that throws after inserting the order but before inserting all items, and assert rollback.
3. Design an outbox row for an order-created event, including a status and an attempt timestamp.

## Review Questions

- What does `commit()` guarantee, and what does it not guarantee about an email sent by PHP?
- Why should a retry re-run the complete transaction?
- How do database constraints complement application validation?
- Which work should be moved outside a short transaction?

## Summary

Transactions group related database changes around one commit decision. Keep them short, use PDO exception handling with rollback, enforce invariants in the schema, make retried commands idempotent, and use an outbox for external effects. Atomic database work is a foundation for reliability, not a promise that every system side effect is atomic.

## References

- [PHP manual: PDO::beginTransaction](https://www.php.net/manual/en/pdo.begintransaction.php)
- [PHP manual: PDO::commit](https://www.php.net/manual/en/pdo.commit.php)
- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL: Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
