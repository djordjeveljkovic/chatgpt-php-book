---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 200
title: Aggregates
slug: aggregates
status: complete
summary: ../../_ai/chapter-summaries/200-aggregates-summary.md
---

# Chapter 200 — Aggregates

## Why This Matters

An aggregate is a consistency boundary around a cluster of domain objects. It has one root through which outside code performs changes, and it protects invariants that must hold together in one transaction. Aggregate design determines what can be changed atomically and how much state a command must load.

An aggregate is not simply every object related by a foreign key. A large graph creates slow transactions, contention, and unclear ownership. Keep the boundary around rules that truly need immediate consistency, and coordinate separate aggregates through events or application services.

## Root and Invariants

The aggregate root owns identity and controls access to child entities and value objects. Outside code references the root and asks it to perform a command; it does not mutate children directly or keep child references for later writes.

Suppose an order must have at least one line before confirmation, and its total must equal the sum of line totals. The root can enforce those rules:

~~~php
<?php

declare(strict_types=1);

final readonly class OrderLine
{
    public function __construct(
        public string $productId,
        public int $quantity,
        public int $unitPriceMinor,
    ) {
        if ($quantity < 1 || $unitPriceMinor < 0) {
            throw new InvalidArgumentException('Invalid order line');
        }
    }

    public function totalMinor(): int
    {
        return $this->quantity * $this->unitPriceMinor;
    }
}

final class Order
{
    /** @var list<OrderLine> */
    private array $lines = [];
    private string $status = 'draft';
    private int $version = 0;

    public function __construct(private readonly string $id)
    {
    }

    public static function reconstitute(string $id, string $status, int $version): self
    {
        if (!in_array($status, ['draft', 'confirmed'], true) || $version < 0) {
            throw new InvalidArgumentException('Invalid persisted order');
        }

        $order = new self($id);
        $order->status = $status;
        $order->version = $version;

        return $order;
    }

    public function id(): string
    {
        return $this->id;
    }

    public function status(): string
    {
        return $this->status;
    }

    public function version(): int
    {
        return $this->version;
    }

    public function addLine(OrderLine $line): void
    {
        if ($this->status !== 'draft') {
            throw new DomainException('Cannot edit a confirmed order');
        }

        $this->lines[] = $line;
    }

    public function confirm(): void
    {
        if ($this->lines === []) {
            throw new DomainException('An order needs a line');
        }

        if ($this->status !== 'draft') {
            throw new DomainException('Order is not draft');
        }

        $this->status = 'confirmed';
    }

    public function totalMinor(): int
    {
        return array_sum(
            array_map(
                static fn (OrderLine $line): int => $line->totalMinor(),
                $this->lines,
            ),
        );
    }
}
~~~

The root does not expose the mutable lines array. A repository or mapper can reconstitute the aggregate through a controlled mechanism, while application code uses commands such as addLine and confirm. The database should still enforce independent constraints and the transaction should protect concurrent commands.

## Choose the Boundary from Invariants

Ask which facts must be true after every committed command:

- Can an order be confirmed without lines?
- Must inventory reservation and order confirmation be atomic?
- Can two users change the same order concurrently?
- Is a child meaningful outside its parent?
- Does the collection have a bounded size and lifecycle?

If inventory belongs to another context or aggregate, do not pull the entire inventory graph into Order to make a convenient object tree. Reserve inventory through an application workflow, use a transaction where the same database owns the invariant, or coordinate with an event and compensation when the boundary is distributed.

## Commands and Queries

Commands should enter through the root and express domain intent: confirm, cancel, add line, or change shipping address. Queries can use a read model or repository query optimized for display, provided they do not mutate the aggregate or bypass authorization.

Do not load an aggregate for every read if the screen needs a projection of thousands of orders. Conversely, do not update a projection directly when the command must enforce the root's invariant. Separate command and query models when the difference in consistency or shape justifies it.

## Concurrency

An aggregate is a concurrency boundary, not automatically a lock. Use optimistic versioning when conflicts are detectable and retry or report a conflict. Use a database lock when the command needs serialized access and the cost is acceptable. Include the version in an atomic update:

~~~php
<?php

declare(strict_types=1);

function saveOrder(PDO $db, Order $order, int $expectedVersion): void
{
    $statement = $db->prepare(
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
~~~

The example assumes line persistence is handled consistently. A real mapper must save the aggregate's lines and version in a transaction. A version check without including every relevant write can still lose changes.

## Size and Performance

The smaller the aggregate, the less state each command locks or loads, but splitting too far can make invariants impossible to enforce locally. Measure query count, row locks, serialization size, and command latency. Keep collections bounded or use a separate process for histories, comments, or audit records that do not participate in the root's invariant.

Do not put an unbounded event log, every customer order, or a large catalog under one aggregate. A root can reference another aggregate by ID and coordinate through a domain or application service.

## Events and Side Effects

An aggregate can record a domain event such as OrderConfirmed after a successful transition. Publishing the event reliably requires an outbox or another transactionally consistent mechanism. Consumers should receive an immutable event with an ID and version and tolerate duplicates if delivery is at least once.

Sending email or calling a provider inside the aggregate couples domain state to network failure. Record the fact and let an application or delivery boundary perform the side effect. The aggregate may decide that an effect is required; it should not own HTTP timeouts or SMTP credentials.

## Reconstitution and Deletion

A repository reconstitutes an aggregate from persistent state without replaying creation commands that would produce a different historical result. Validate persisted invariants and handle old versions through migrations or explicit compatibility code. Do not silently accept corrupt state.

Deletion must respect references, audit, legal retention, and events. A command such as archive can preserve identity and history while preventing new mutations. If child rows are deleted together, define database cascade and transaction behavior rather than relying on object destruction.

## Testing and Operations

Test each invariant through root commands, including invalid transitions and boundary values. Test repository mapping and optimistic conflicts against the real database. Test concurrent commands, duplicate messages, outbox publication, and authorization at the application boundary.

Observe aggregate load size, command latency, version conflicts, transaction duration, lock waits, and event lag. These metrics reveal an oversized boundary or a contention hotspot. Log aggregate IDs with tenant and correlation context without exposing sensitive line data.

## Common Mistakes

- Treating every related database row as one aggregate.
- Letting callers mutate child entities directly.
- Loading huge collections for a small command.
- Using an aggregate as a distributed transaction.
- Publishing events without an outbox or duplicate policy.
- Assuming a version field prevents conflicts without atomic writes.
- Putting network credentials and side effects in domain objects.

## Senior Engineer Thinking

An aggregate defines which invariants and state changes commit together. Keep one root, expose intent-revealing commands, keep the boundary small enough for expected concurrency, and coordinate other aggregates through explicit workflows and events. Use versioning or locks deliberately, and observe load, contention, and delivery behavior.

## Exercises

1. Define the aggregate boundary for an order, inventory reservation, and shipment workflow.
2. Add an optimistic version to the Order example and test two concurrent confirmations.
3. Design an outbox event for OrderConfirmed with duplicate delivery handling.
4. Identify an unbounded child collection and decide whether it belongs inside the aggregate.

## Review Questions

1. What makes an aggregate a consistency boundary?
2. Why must outside code use the root?
3. When should two related objects be separate aggregates?
4. What does optimistic versioning protect, and what can it miss?
5. Why does an aggregate not make external side effects atomic?

## Summary

Aggregates group state and behavior that must obey invariants in one consistency boundary. Use a root to expose commands, keep collections bounded, choose locks or versions for concurrency, separate read projections, and coordinate other aggregates through application workflows and reliable events. Aggregates protect domain rules but do not replace database constraints or distributed recovery.

## References

- [Martin Fowler: Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- [Eric Evans: Domain-Driven Design](https://www.domainlanguage.com/ddd/)
- [Vaughn Vernon: Effective Aggregate Design](https://www.dddcommunity.org/library/vernon_2011/)
- [PHP PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)
