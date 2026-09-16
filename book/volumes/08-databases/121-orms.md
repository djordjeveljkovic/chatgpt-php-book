---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 121
title: ORMs
slug: orms
status: complete
summary: ../../_ai/chapter-summaries/121-orms-summary.md
---

# Chapter 121 — ORMs

## Why This Matters

An object-relational mapper (ORM) lets PHP code work with objects while the database works with rows and sets. That translation can remove repetitive mapping code and make a domain model pleasant to use. It can also hide query count, transaction boundaries, locking, null semantics, and the amount of data being hydrated.

An ORM is a persistence tool, not a replacement for SQL knowledge. The engineer still chooses query shape, indexes, transaction scope, consistency, and failure behavior. This chapter uses ORM concepts that apply broadly, with Doctrine terminology where it names a common implementation. Framework-specific APIs and generated SQL must be checked against the installed version.

## The Mapping Boundary

Suppose the database stores an order and its status as columns:

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    total_cents INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

The PHP model can expose value objects and behavior rather than leaking storage details:

```php
<?php

declare(strict_types=1);

enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Cancelled = 'cancelled';
}

final class Order
{
    public function __construct(
        private readonly int $id,
        private readonly int $customerId,
        private OrderStatus $status,
        private readonly int $totalCents,
    ) {
    }

    public function markPaid(): void
    {
        if ($this->status !== OrderStatus::Pending) {
            throw new DomainException('Only pending orders can be paid');
        }

        $this->status = OrderStatus::Paid;
    }
}
```

The mapper must turn `status` into `OrderStatus` and preserve the invariant that only valid transitions are possible. Whether the ORM constructs this object through reflection, a proxy, or a generated hydrator is implementation-specific. The important boundary is that database state and domain state have different representations and lifecycles.

## Identity Maps and Unit of Work

Many ORMs keep an identity map: within one persistence context, two loads of the same database identity return the same in-memory object. A unit of work tracks new, changed, and removed objects and calculates SQL during a flush. These features make a graph of related objects convenient to edit, but they also mean that `flush()` is a meaningful operation with cost and failure modes.

```php
$order = $entityManager->find(Order::class, $orderId);
$order->markPaid();

$entityManager->flush(); // SQL and transaction policy meet here.
```

Do not infer that `flush()` is a transaction commit. Some ORMs wrap a flush in a transaction for a single operation; some applications own the transaction explicitly; configuration and version matter. For a multi-step business operation, establish the transaction boundary deliberately and verify generated SQL and exception behavior.

Long-running commands must clear or detach managed entities after each batch. Otherwise the identity map retains every object and dirty checking grows with the process. This is the ORM form of the large-dataset problem described in [Chapter 120](120-large-datasets.md).

## Hydration Is a Query Decision

Hydrating a full entity graph is more expensive than selecting scalar values. Full entities fit behavior-rich workflows that modify a small set of aggregates. Scalar or array results fit reports and lists. A dedicated read model fits complex screens where one shape is assembled from several tables.

Loading relationships can produce the N+1 query problem:

```php
$orders = $repository->findBy(['status' => OrderStatus::Pending]);

foreach ($orders as $order) {
    echo $order->customer()->name(); // May issue one query per order.
}
```

The fix is not always “eager load everything.” Fetch the relationship in the query when this use case needs it, batch-load related identities, or project the required columns. Eagerly loading a large collection can create row multiplication and a very large result to hydrate. Inspect query count, row count, and memory for the actual endpoint.

## Lazy Loading and Hidden I/O

Lazy properties and proxies make an object look local while accessing it can perform I/O. That is convenient inside a controlled application service and surprising in a serializer, template, logger, or loop. A detached object may also fail when a lazy association needs a closed entity manager.

Set an explicit boundary: repositories or query services load the data needed by the application operation, and presentation code consumes an already-defined view. Disable lazy loading where hidden queries would be a latency risk, or use a profiler/test assertion that detects query-count regressions.

## Transactions and Aggregate Boundaries

A persistence context can span many objects, but a transaction should cover one business operation whose state changes must commit together. Do not expose an entity manager throughout the application and let arbitrary callers decide when to flush. A service can own the unit of work:

```php
final class CapturePayment
{
    public function __construct(
        private OrderRepository $orders,
        private EntityManager $entityManager,
    ) {
    }

    public function __invoke(int $orderId): void
    {
        $this->entityManager->wrapInTransaction(function () use ($orderId): void {
            $order = $this->orders->get($orderId);
            $order->markPaid();
            $this->entityManager->flush();
        });
    }
}
```

The helper name is illustrative; use the installed ORM's API. If payment is an external call, do not hold a database transaction open while waiting for the provider. Record an intent, commit, call the provider through an idempotent workflow, and reconcile the result as the business protocol requires.

## ORM Queries and Concurrency

A repository method such as `findRecentForCustomer()` should have a known ordering, projection, relation strategy, and index. Log SQL with bind values redacted, inspect plans for important paths, and make query count a testable property where practical. Raw SQL is valid for window functions, bulk updates, vendor-specific features, and read models; keep it in a named query service and retain parameterization.

An entity loaded at time A can be flushed at time B after another transaction changed the row. Use optimistic version fields when lost updates must be detected, or explicit row locks when the operation requires serialization. An ORM does not remove database concurrency behavior; it may postpone the update until flush, making a conflict less obvious.

Do not use an entity object as a durable cache across requests. PHP-FPM workers do not share ordinary object memory, and even a long-running worker can hold stale managed state. Reload or clear the persistence context according to the operation's consistency needs.

## Common Mistakes

- Treating ORM convenience as evidence that a query is efficient.
- Triggering N+1 queries through lazy relationships in a loop or serializer.
- Eager-loading every association and hydrating a huge object graph.
- Assuming `flush()` has the transaction semantics the use case needs.
- Keeping every managed entity in a long-running import.
- Returning entities directly from an API and exposing lazy I/O or internal fields.
- Hiding a bulk operation inside per-entity updates.
- Retrying a failed flush without checking which external effects already happened.

## Senior Engineer Thinking

Use an ORM where object identity, domain behavior, and small aggregate edits justify its mapping and unit-of-work costs. Use projections, SQL, or a query builder where the use case is read-heavy, set-oriented, or vendor-specific. The deciding question is whether the resulting query, transaction, memory, and concurrency behavior fits the operation.

## Exercises

1. Map an order status column to a PHP backed enum and define valid state transitions.
2. Take a lazy relationship loop and redesign it to avoid N+1 queries. Compare eager loading, batch loading, and a projection.
3. Design a transaction boundary for an operation that changes an order and writes an outbox record.
4. Explain how an optimistic version field detects a stale entity during ORM flush.

## Review Questions

1. What do an identity map and a unit of work provide?
2. Why can lazy loading create a performance bug without changing application-level code?
3. Why is `flush()` not automatically the same as committing a business transaction?
4. When is a scalar projection a better fit than a fully hydrated entity?
5. How should a long-running ORM process control its managed identity map?

## Summary

ORMs translate between relational rows and PHP objects, while identity maps and units of work add useful lifecycle behavior and hidden cost. Choose entity graphs, projections, query services, or raw SQL per use case. Own transaction boundaries, detect N+1 and stale-object problems, clear long-running contexts, and inspect the SQL the ORM emits.

## References

- [Doctrine ORM: Working with objects](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html)
- [Doctrine ORM: DQL](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/dql-doctrine-query-language.html)
- [PHP Manual: Enumerations](https://www.php.net/manual/en/language.enumerations.php)
- [PostgreSQL: Explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
