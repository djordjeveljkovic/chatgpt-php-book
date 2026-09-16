---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 168
title: Fakes
slug: fakes
status: complete
summary: ../../_ai/chapter-summaries/168-fakes-summary.md
---

# Chapter 168 — Fakes

A fake is a working but simplified implementation of a real collaborator. An in-memory repository, local filesystem, deterministic clock, or test queue can exercise more of the application than a stub while remaining fast and controlled.

A fake has behavior. That makes it valuable and dangerous: it can reveal realistic interactions, but a fake with different semantics from production creates false confidence. Keep the fake small, document its differences, and cover the shared contract with integration tests against the real implementation.

## Why this matters

A service that stores an order and later reads it needs more than a predetermined response. The test may need to verify that the write is visible to the read, that duplicate IDs are rejected, and that a missing record behaves consistently. An in-memory fake can model those rules without requiring a database for every unit-level test.

Start with a narrow repository interface:

```php
<?php

declare(strict_types=1);

final readonly class Order
{
    public function __construct(
        public string $id,
        public int $customerId,
        public int $totalCents,
    ) {
    }
}

interface OrderRepository
{
    public function save(Order $order): void;

    public function find(string $id): ?Order;
}
```

A fake implements the useful semantics explicitly:

```php
final class InMemoryOrderRepository implements OrderRepository
{
    /** @var array<string, Order> */
    private array $orders = [];

    public function save(Order $order): void
    {
        if (isset($this->orders[$order->id])) {
            throw new LogicException('Duplicate order ID');
        }

        $this->orders[$order->id] = $order;
    }

    public function find(string $id): ?Order
    {
        return $this->orders[$id] ?? null;
    }
}
```

A test can exercise a complete interaction:

```php
public function testSavedOrderCanBeReadBack(): void
{
    $repository = new InMemoryOrderRepository();
    $order = new Order('ord-1', 42, 2500);

    $repository->save($order);

    self::assertSame($order, $repository->find('ord-1'));
}
```

The fake's duplicate rule is a deliberate contract. If the production database enforces uniqueness, the fake should reject duplicates too. If the fake silently overwrites them, tests may approve behavior production cannot provide.

## Useful fake boundaries

Fakes work well for collaborators whose behavior can be represented locally:

- repositories with a small data model;
- queues that retain published messages;
- clocks with an explicitly controlled time;
- object stores backed by a temporary directory;
- payment gateways that record a deterministic authorization result;
- feature-flag providers with configured evaluations.

A fake queue can model publication and consumption without pretending to be a distributed broker:

```php
final class InMemoryQueue
{
    /** @var list<array{type: string, payload: array<string, mixed>}> */
    private array $messages = [];

    /** @param array<string, mixed> $payload */
    public function publish(string $type, array $payload): void
    {
        $this->messages[] = ['type' => $type, 'payload' => $payload];
    }

    /** @return list<array{type: string, payload: array<string, mixed>}> */
    public function messages(): array
    {
        return $this->messages;
    }
}
```

The fake does not model visibility timeouts, redelivery, ordering across workers, or broker durability. Those properties need contract and integration tests against the actual queue or a behaviorally faithful test environment.

## Keeping fakes honest

A fake should have one source of truth for its rules where possible. Avoid copying production implementation line by line; that merely creates two versions to maintain. Instead, express the domain contract in shared tests:

```php
/** @return iterable<string, OrderRepository> */
public function repositoryImplementations(): iterable
{
    yield 'in memory' => [new InMemoryOrderRepository()];
    yield 'database' => [$this->databaseRepository()];
}

/**
 * @dataProvider repositoryImplementations
 */
public function testRepositoryContract(OrderRepository $repository): void
{
    $order = new Order('ord-1', 42, 2500);
    $repository->save($order);

    self::assertSame($order, $repository->find('ord-1'));
}
```

The exact data-provider syntax depends on the PHPUnit version. The principle is to run the same observable contract against the fake and the real adapter. Add cases for missing records, duplicate writes, transaction behavior, uniqueness, filtering, ordering, and authorization where they belong.

Record fake limitations in the test documentation. For example, a file fake may run on a case-sensitive Linux filesystem while production runs on a different filesystem. A database fake may not reproduce collation, isolation, constraints, or query performance.

## State, isolation, and reset

A fake is mutable test state. Create a fresh instance per test or reset it in a framework-supported setup hook. Sharing a fake across tests creates order dependence and hides missing cleanup. Avoid global singletons and static storage.

Bound the fake's resource use. A test that publishes millions of messages or stores unbounded files should fail for the same reason a production service would: resource limits are part of behavior. Add explicit failure injection for disk-full, duplicate, timeout, and unavailable states rather than making the fake unrealistically successful.

## Testing and operations

Fakes are excellent for fast service tests and workflow examples. They do not prove SQL, indexes, network TLS, broker delivery, filesystem permissions, or production configuration. Keep real-adapter integration tests small but meaningful, and run contract tests in CI and before dependency upgrades.

When a fake diverges, fix the contract test or remove the fake rather than adding a special case that only makes one test pass. The purpose of a fake is to provide useful behavior at lower cost, not to disguise an incompatible production implementation.

## Exercises

1. Extend the repository fake with a `delete()` operation and define whether deleting a missing order is an error or a no-op.
2. Add a contract test for duplicate IDs and run it against both the fake and a real database repository.
3. Add failure injection to the queue fake for publication errors and verify the service's recovery policy.

## Review questions

- What makes a fake different from a stub?
- Which production semantics must a repository fake model?
- Why can a fake queue not prove broker redelivery behavior?
- How do shared contract tests prevent fake drift?
- Why should each test receive a fresh fake instance?

## References

- [PHPUnit documentation: Test Doubles](https://docs.phpunit.de/en/11.5/test-doubles.html)
- [Martin Fowler: Test Double](https://martinfowler.com/bliki/TestDouble.html)
- [PHPUnit documentation: Data providers](https://docs.phpunit.de/en/11.5/writing-tests-for-phpunit.html#data-providers)
