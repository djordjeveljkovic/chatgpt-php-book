---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 123
title: Database vs PHP Responsibilities
slug: database-vs-php-responsibilities
status: complete
summary: ../../_ai/chapter-summaries/123-database-vs-php-responsibilities-summary.md
---

# Chapter 123 — Database vs PHP Responsibilities

## Why This Matters

An application is reliable when each invariant is enforced at the narrowest boundary that can prove it. A database can make uniqueness and referential integrity atomic across all writers. PHP can express a workflow, call external services, and present useful validation errors. Putting a rule in the wrong layer creates races, duplicated logic, excessive data transfer, or a database that cannot protect its own state.

There is no universal rule that every decision belongs in PHP or every computation belongs in SQL. The boundary follows the invariant, the data volume, the transaction, the ownership of the rule, and the operational capabilities of the team.

## Facts the Database Must Protect

If a fact must be true for every writer, enforce it in the schema:

- A primary key identifies one row.
- A foreign key references an existing row when the relationship is required.
- A unique constraint prevents duplicate identities.
- A `NOT NULL` constraint prevents missing required data.
- A check constraint limits values where the database can express the invariant.

This application-side check is not sufficient:

```php
if ($repository->findByEmail($email) !== null) {
    throw new DomainException('Email is already registered');
}

$repository->insert($email);
```

Two requests can both observe absence. The database must be the final arbiter:

```sql
ALTER TABLE users
    ADD CONSTRAINT users_email_unique UNIQUE (email_normalized);
```

PHP can pre-check to produce a friendly message, but it must handle the constraint violation as a normal race outcome. [Chapter 118](118-concurrency.md) explains this pattern in detail.

## Business Policy Belongs in the Application

A database can enforce that an order has a valid status value. It usually should not own the whole meaning of “a paid order can be refunded only within 30 days unless the account is premium and settlement has not started.” That policy combines actors, clocks, external systems, and product decisions. Put the decision in a tested application service, then use a transaction and database constraints to protect the resulting state transition.

```php
final class RefundOrder
{
    public function __construct(
        private OrderRepository $orders,
        private Clock $clock,
    ) {
    }

    public function __invoke(int $orderId, Account $account): void
    {
        $order = $this->orders->get($orderId);
        if (!$order->canBeRefundedAt($this->clock->now(), $account)) {
            throw new DomainException('Order is not refundable');
        }

        $order->refund();
        $this->orders->save($order);
    }
}
```

The database still enforces values, relationships, and concurrent transitions. PHP owns the policy because it is the application's language and can be tested against explicit examples. If the policy is shared by multiple independent services, move the rule to a shared service or durable protocol rather than assuming one PHP code path is universal.

## Push Work Down When the Data Is There

Filtering, joining, grouping, and aggregating belong close to the data when the database can use indexes and avoid transferring irrelevant rows:

```sql
SELECT customer_id, SUM(total_cents) AS total_cents
FROM orders
WHERE created_at >= :start AND created_at < :end
GROUP BY customer_id;
```

Fetching every order into PHP and summing it creates memory, network, and latency cost. It also risks returning an answer from a moving set of rows unless the query's consistency boundary is understood. Use PHP when the transformation needs external calls, rich domain behavior, a library unavailable in the database, or a deliberately staged batch. State the reason rather than treating “SQL is faster” as a complete argument.

The database is also the right place for atomic predicates:

```sql
UPDATE inventory
SET available = available - :quantity
WHERE sku = :sku AND available >= :quantity;
```

PHP checks the affected-row count and chooses the user-facing outcome. The database checks the current value while serializing competing updates. Splitting this into a read in PHP and a later write loses the invariant under concurrency.

## Validation Has Two Layers

Request validation belongs in PHP because it can report all user-facing errors, normalize input, and apply context-sensitive rules. Storage validation belongs in the database because another process, migration, script, or old application version can write the same tables.

For a price, PHP can reject malformed input and convert a decimal string into integer cents. The database can require a nonnegative integer:

```sql
ALTER TABLE products
    ADD CONSTRAINT products_price_nonnegative CHECK (price_cents >= 0);
```

Do not rely on a PHP type declaration to validate a database column. `string`, `int`, and `DateTimeImmutable` describe an in-process value; they do not constrain direct SQL, imports, or concurrent writers. Conversely, a database constraint is a poor user-interface validator when it cannot express which field or correction the user needs.

## Transactions Cross the Boundary

The transaction should cover the database changes that must commit together. PHP chooses the business operation and calls the database transaction; the database provides atomicity and isolation for its writes.

```php
$pdo->beginTransaction();
try {
    $orderId = createOrder($pdo, $input);
    addOrderLines($pdo, $orderId, $input->lines);
    writeOutboxEvent($pdo, $orderId);
    $pdo->commit();
} catch (Throwable $exception) {
    if ($pdo->inTransaction()) {
        $pdo->rollBack();
    }
    throw $exception;
}
```

Do not put an email, payment request, or remote HTTP call inside this transaction and assume rollback can undo it. Use an outbox, idempotent external operation, or a compensating workflow. The database can commit its durable fact; PHP and a worker coordinate the external side effect afterward.

## Triggers, Stored Procedures, and Generated Values

Triggers and stored procedures can enforce or centralize rules close to data. They are useful when writes come from many applications, when an audit row must be created for every direct update, or when a set-based operation is naturally expressed in the database. They can also surprise application developers, complicate local tests, and make write cost or ordering less visible.

Use a trigger when the guarantee must survive every write path and the team can observe and migrate it. Use a stored procedure when the database operation is a deliberate interface with clear ownership and permissions. Document side effects, generated values, error codes, and transaction expectations. Do not hide a product workflow in a trigger merely to avoid designing an application service.

## Caching and Derived Data

The database is usually the source of truth for durable business state. PHP or a cache can hold a derived view, but cache invalidation and stale reads become part of the contract. Never use a process-local PHP array as a cross-worker authority for inventory, permissions, or uniqueness.

If a value is derived and expensive, consider a materialized view, summary table, cache, or asynchronous projection. Define who refreshes it, what staleness is acceptable, and how a rebuild works. A cache hit should not bypass an authorization rule that depends on current state unless the cached policy has an explicit freshness guarantee.

## A Decision Table

| Concern | Primary owner | Database support | PHP responsibility |
| --- | --- | --- | --- |
| Unique identity | Database | Unique constraint | Map conflict to a useful result |
| Required relationship | Database | Foreign key and nullability | Choose lifecycle and error handling |
| User input shape | PHP | Column type and constraints | Normalize and report validation errors |
| Aggregate query | Database | Filter, join, group, index | Choose projection and present result |
| Multi-step policy | PHP | Transaction, constraints, locks | Decide and coordinate the workflow |
| Atomic stock transition | Database | Guarded update | Interpret affected rows |
| External side effect | PHP/service | Outbox state | Retry idempotently and reconcile |

This is a starting point, not a division of authority by technology. If a rule must hold when a maintenance script writes directly to the table, the database needs a constraint or controlled write interface. If a rule changes with product policy and includes an external response, PHP needs an explicit service and tests.

## Common Mistakes

- Performing a uniqueness or balance check in PHP without a database guard.
- Loading millions of rows into PHP to perform a filter or aggregate the database can do.
- Putting user-facing validation messages in a trigger with no mapping strategy.
- Calling an external service inside a transaction and expecting rollback to undo it.
- Treating cache state or one PHP worker's memory as the source of truth.
- Adding triggers that silently change writes without documenting their side effects.
- Duplicating a constraint in many services without a single authoritative definition.
- Letting a database constraint become the only validation path for an interactive form.

## Senior Engineer Thinking

Ask what must always be true, who can write the data, and which operation can make the decision atomically. Put durable cross-writer facts in the database, policy and orchestration in PHP, set-oriented work near the data, and external effects behind an explicit workflow. Then test the boundary under concurrency, failure, and direct data access rather than only through the happy-path controller.

## Exercises

1. Move an application-side uniqueness check into a database constraint and design its duplicate-error handling.
2. Split a refund workflow into PHP policy, a database transaction, and an external payment step.
3. Compare summing a million rows in PHP with a database aggregate. Include consistency, transfer cost, and observability.
4. Decide whether an audit requirement belongs in application code, a trigger, or both. Document the writers and failure modes.

## Review Questions

1. Which rules should survive every writer to a table?
2. Why are user-facing validation and database constraints both needed?
3. What cannot a database rollback undo once PHP has called an external service?
4. When is a trigger an appropriate enforcement boundary?
5. How should a derived cache state its freshness and rebuild contract?

## Summary

Put cross-writer facts such as identity, relationships, nullability, and atomic guarded transitions in the database. Put context-rich product policy, input feedback, orchestration, and external side effects in PHP or a service around it. Push set-oriented filtering and aggregation toward the data, own transaction boundaries explicitly, and use constraints, outboxes, locks, and idempotency to make the boundary reliable.

## References

- [PostgreSQL: Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [MySQL: InnoDB and foreign keys](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html)
- [PHP Manual: PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)
- [Martin Fowler: Transactional Outbox](https://martinfowler.com/articles/patterns-of-distributed-systems/transactional-outbox.html)
