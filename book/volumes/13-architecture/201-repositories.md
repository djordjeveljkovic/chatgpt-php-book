---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 201
title: Repositories
slug: repositories
status: complete
summary: ../../_ai/chapter-summaries/201-repositories-summary.md
---

# Chapter 201 — Repositories

## Why This Matters

A Repository is an application-facing collection boundary for domain objects. It answers questions such as “load this order for update” or “find the active subscription for this account” without forcing domain code to know SQL, HTTP, ORM sessions, or storage layout.

A repository is not automatically a generic CRUD wrapper. Its contract must state identity scope, missing data, ordering, consistency, locking, pagination, and failure behavior. A repository that hides a query cost or returns a stale object can make a simple method operationally dangerous.

## Design the Port from Use Cases

Start with the operations a use case needs. Keep the interface small and domain-oriented:

~~~php
<?php

declare(strict_types=1);

interface OrderRepository
{
    public function getForUpdate(string $orderId, int $tenantId): ?Order;

    public function save(Order $order, int $expectedVersion): void;

    /** @return list<OrderSummary> */
    public function findOpenForCustomer(int $tenantId, int $customerId): array;
}
~~~

The port says that an update is tenant-scoped and version-aware. It does not expose a PDO statement or a vendor query builder. A missing aggregate can be represented by null when the caller can decide whether it is a not-found or authorization response; an exception can be appropriate when the absence violates an invariant.

Avoid methods such as findAll, saveAny, or query(string $sql) unless the boundary explicitly owns that generic behavior. Broad methods invite unbounded queries, mass assignment, and callers that depend on persistence details.

## Repository and Aggregate Boundaries

Repository methods should usually load and save an aggregate root, not arbitrary child entities. The aggregate root controls invariants and the repository controls its persistence boundary. A child lookup may be a reporting query or a separate read model, but it should not allow a caller to bypass the root.

A repository does not decide every business rule. It can scope by tenant, apply a lock, and enforce an optimistic version condition. The domain decides whether an order can be confirmed; the application service decides which actor may issue the command.

## A PDO Adapter

An adapter translates the domain port into database queries and maps rows explicitly:

~~~php
<?php

declare(strict_types=1);

final class PdoOrderRepository implements OrderRepository
{
    public function __construct(private PDO $db)
    {
    }

    public function getForUpdate(string $orderId, int $tenantId): ?Order
    {
        $statement = $this->db->prepare(
            'SELECT id, tenant_id, status, version
             FROM orders
             WHERE id = :id AND tenant_id = :tenant
             FOR UPDATE',
        );
        $statement->execute(['id' => $orderId, 'tenant' => $tenantId]);
        $row = $statement->fetch(PDO::FETCH_ASSOC);

        return $row === false ? null : $this->mapOrder($row);
    }

    public function save(Order $order, int $expectedVersion): void
    {
        $statement = $this->db->prepare(
            'UPDATE orders
             SET status = :status, version = version + 1
             WHERE id = :id AND version = :version',
        );
        $statement->execute([
            'status' => $order->status(),
            'id' => $order->id(),
            'version' => $expectedVersion,
        ]);

        if ($statement->rowCount() !== 1) {
            throw new DomainException('Order changed concurrently');
        }
    }

    public function findOpenForCustomer(int $tenantId, int $customerId): array
    {
        $statement = $this->db->prepare(
             'SELECT id, status FROM orders
             WHERE tenant_id = :tenant AND customer_id = :customer
               AND status IN (\'draft\', \'confirmed\')
             ORDER BY id',
        );
        $statement->execute([
            'tenant' => $tenantId,
            'customer' => $customerId,
        ]);

        $orders = [];
        while ($row = $statement->fetch(PDO::FETCH_ASSOC)) {
            $orders[] = new OrderSummary((string) $row['id'], (string) $row['status']);
        }

        return $orders;
    }

    /** @param array<string, mixed> $row */
    private function mapOrder(array $row): Order
    {
        return Order::reconstitute(
            (string) $row['id'],
            (string) $row['status'],
            (int) $row['version'],
        );
    }
}
~~~

The SQL uses PostgreSQL-style row locking; other databases require their own locking syntax and transaction behavior. The caller must open a transaction around getForUpdate and save. Never claim that a method locks unless the database transaction and isolation semantics make that true.

The query scopes tenant ID in the database, uses parameters, and maps only known fields. A production adapter should also handle deleted or archived records, schema versions, and database exceptions according to the port's contract.

## Transactions and Unit of Work

A repository should not silently begin and commit a transaction around every method when a use case needs several repositories to commit together. Let the application service or unit-of-work boundary own the transaction:

~~~php
<?php

declare(strict_types=1);

final class ConfirmOrder
{
    public function __construct(
        private OrderRepository $orders,
        private Transaction $transaction,
    ) {
    }

    public function run(string $orderId, int $tenantId, int $version): void
    {
        $this->transaction->run(function () use ($orderId, $tenantId, $version): void {
            $order = $this->orders->getForUpdate($orderId, $tenantId);
            if ($order === null) {
                throw new DomainException('Order unavailable');
            }

            $order->confirm();
            $this->orders->save($order, $version);
        });
    }
}
~~~

The version supplied by the caller should normally come from the loaded aggregate, not an untrusted request field. The example is abbreviated to emphasize the boundary. A real service would load the current version, authorize the actor, and include all aggregate writes and outbox records in the transaction.

## Queries, Pagination, and Read Models

A repository query must define ordering and limits. Do not return an unbounded collection from findOpenForCustomer in a high-volume system. Add a cursor or a dedicated query/read-model port when the screen needs pagination, projections, or joins that do not belong to the aggregate.

Read models can have different repositories from command-side aggregates. A dashboard repository may return immutable OrderSummary values and tolerate bounded eventual consistency. It must not be used to make a financial or authorization decision without a current consistency check.

## Caching and Identity

Caching inside a repository changes freshness, invalidation, and authorization behavior. Include tenant and authorization scope in cache keys, define a version or invalidation policy, and do not return a mutable cached aggregate across requests or jobs. A process-local identity map can prevent duplicate instances during one unit of work, but it must be cleared at the boundary.

A cache miss, stale read, database outage, and missing row are different outcomes. Represent or map them deliberately so callers do not turn infrastructure failure into “not found.”

## Testing Repository Contracts

Define shared contract tests for every adapter:

- saving and loading preserves identity, state, value objects, and version;
- tenant scoping never returns another tenant's aggregate;
- missing rows have the documented result;
- duplicate or invalid data produces a defined failure;
- optimistic conflicts reject stale writes;
- row locks and transaction boundaries behave on the real engine;
- pagination ordering and cursors are stable;
- deleted or archived records follow policy.

An in-memory fake is useful for application unit tests but cannot prove SQL, constraints, isolation, indexes, or lock behavior. Run integration tests against the production database engine and schema migrations.

## Common Mistakes

- Creating a generic repository with unbounded query methods.
- Loading child entities outside the aggregate root.
- Hiding transactions inside individual repository calls.
- Claiming a row is locked without an open compatible transaction.
- Omitting tenant scope from queries and cache keys.
- Treating stale, missing, and unavailable data as the same result.
- Using an in-memory fake as proof of database behavior.
- Returning mutable cached aggregates across job boundaries.

## Senior Engineer Thinking

A repository is a contract between application policy and persistence. Design it from use cases and aggregate boundaries, make missing, locking, version, pagination, and failure semantics explicit, keep transactions at the application boundary, and verify adapters against the real database. Keep read models and command repositories separate when their consistency and shape differ.

## Exercises

1. Design an OrderRepository contract with tenant scope, optimistic version, pagination, and archived-order policy.
2. Implement a PDO adapter and test it against the real database engine for stale writes and row locks.
3. Compare an aggregate repository with a dashboard query repository and list their different consistency guarantees.
4. Add tenant-aware caching and test that one tenant cannot observe another's aggregate.

## Review Questions

1. What makes a repository domain-oriented rather than a generic CRUD wrapper?
2. Which component should usually own a multi-repository transaction?
3. Why must row-locking claims include transaction semantics?
4. How do read-model repositories differ from aggregate repositories?
5. Why can an in-memory fake not prove database behavior?
6. Which scope belongs in repository queries and cache keys?

## Summary

Repositories provide domain-oriented persistence contracts around aggregate and read-model boundaries. Define missing, locking, version, pagination, transaction, and failure semantics; map rows explicitly in adapters; scope tenants in queries and caches; keep transactions at the application boundary; and test adapters against the real database while using fakes only for focused application tests.

## References

- [Martin Fowler: Repository](https://martinfowler.com/eaaCatalog/repository.html)
- [Martin Fowler: Data Mapper](https://martinfowler.com/eaaCatalog/dataMapper.html)
- [Eric Evans: Domain-Driven Design](https://www.domainlanguage.com/ddd/)
- [PHP PDO prepared statements](https://www.php.net/manual/en/pdo.prepare.php)
- [PHP PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)
