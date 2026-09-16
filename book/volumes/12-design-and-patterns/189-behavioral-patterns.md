---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 189
title: Behavioral Patterns
slug: behavioral-patterns
status: complete
summary: ../../_ai/chapter-summaries/189-behavioral-patterns-summary.md
---

# Chapter 189 — Behavioral Patterns

## Why This Matters

Behavioral patterns organize collaboration, decisions, state transitions, and communication between objects. They are useful when an operation has interchangeable rules, a request must be queued or replayed, a state machine has explicit transitions, or several consumers should react without the caller knowing each one.

A pattern name is not a design. Start with the behavior that varies, the state that must remain valid, and the failure or transaction boundary. Add an object only when it makes that behavior clearer, testable, or independently replaceable.

## Strategy

The Strategy pattern moves an interchangeable algorithm behind a small interface. A pricing rule, authorization policy, retry policy, or serializer can vary without a conditional tree spreading through the caller:

~~~php
<?php

declare(strict_types=1);

interface ShippingRate
{
    public function quote(int $weightGrams, string $destination): int;
}

final class StandardShipping implements ShippingRate
{
    public function quote(int $weightGrams, string $destination): int
    {
        return 500 + intdiv($weightGrams, 1000) * 100;
    }
}

final class ExpressShipping implements ShippingRate
{
    public function quote(int $weightGrams, string $destination): int
    {
        return 1200 + intdiv($weightGrams, 1000) * 250;
    }
}

final class ShippingService
{
    public function __construct(private ShippingRate $rate)
    {
    }

    public function quote(int $weightGrams, string $destination): int
    {
        if ($weightGrams <= 0) {
            throw new InvalidArgumentException('Weight must be positive');
        }

        return $this->rate->quote($weightGrams, $destination);
    }
}
~~~

The interface should describe the decision the caller needs, not expose every provider option. Select a strategy in the composition root or from a validated configuration. If the choice is a simple stable branch with no independent testing or change, a match expression is clearer.

## Command

A Command object represents an operation and its data. It can be validated, authorized, queued, retried, logged with a safe identifier, or deduplicated. The handler owns execution:

~~~php
<?php

declare(strict_types=1);

final readonly class CapturePayment
{
    public function __construct(
        public int $orderId,
        public int $amountCents,
        public string $idempotencyKey,
    ) {
        if ($amountCents <= 0 || $idempotencyKey === '') {
            throw new InvalidArgumentException('Invalid capture command');
        }
    }
}

interface CommandHandler
{
    public function handle(CapturePayment $command): void;
}
~~~

A command is not automatically a message. An in-process command can rely on the current transaction; a queued command needs a schema version, serialization policy, retry behavior, authorization context, and idempotency. Persist an outbox record when a database change and publication must be coordinated. Never put raw secrets in a command log.

## State

The State pattern makes legal behavior depend on an explicit state object or transition model. It is useful when a resource has many operations whose rules vary by state. For a small state machine, an enum and a transition method may be enough:

~~~php
<?php

declare(strict_types=1);

enum OrderState: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Cancelled = 'cancelled';

    public function transitionTo(self $next): self
    {
        $allowed = match ($this) {
            self::Pending => [$this::Paid, $this::Cancelled],
            self::Paid => [],
            self::Cancelled => [],
        };

        if (!in_array($next, $allowed, true)) {
            throw new DomainException('Invalid order transition');
        }

        return $next;
    }
}
~~~

For complex states, separate state classes can keep each policy cohesive. Whichever representation is used, enforce the transition atomically with the persistence update. An in-memory state check cannot prevent two requests from both paying an order; use a version, lock, or conditional update.

## Observer and Events

An observer or event subscriber lets a publisher notify interested consumers without direct calls to each one. This can reduce compile-time coupling, but it introduces delivery, ordering, duplicate, schema, and failure semantics:

~~~php
<?php

declare(strict_types=1);

final readonly class OrderPlaced
{
    public function __construct(
        public int $orderId,
        public int $customerId,
        public int $schemaVersion = 1,
    ) {
    }
}

interface OrderPlacedListener
{
    public function handle(OrderPlaced $event): void;
}

final class OrderEventPublisher
{
    /** @param list<OrderPlacedListener> $listeners */
    public function __construct(private array $listeners)
    {
    }

    public function publish(OrderPlaced $event): void
    {
        foreach ($this->listeners as $listener) {
            $listener->handle($event);
        }
    }
}
~~~

In-process observers run in the publisher's request and can make a transaction slow or fail. A durable event or outbox worker has different guarantees. Consumers must be idempotent, and event payloads must not expose data merely because a subscriber might want it. Version and document schemas.

## Chain of Responsibility

A chain passes a request through handlers until one handles it or the chain ends. Middleware, validation pipelines, and approval rules often use this shape. The chain must define ordering, short-circuit behavior, and whether a handler may transform the request.

Use a chain when handlers are independently ordered or enabled. For a small fixed sequence, ordinary code is easier to read. A security chain should fail closed when a required handler is missing; a logging handler should not accidentally turn an authorization denial into success.

## Template Method and Iteration

Template Method fixes an algorithm's sequence while allowing selected steps to vary, often through inheritance. Prefer composition and Strategy when variation does not need protected lifecycle hooks; inheritance couples subclasses to the base class's order and state. An abstract base class is appropriate when the invariant sequence itself is the contract and subclasses provide narrow operations.

An Iterator exposes sequential traversal without exposing collection representation. PHP's iterable types and generators already provide this capability for many cases. A generator can stream rows and bound memory, but the underlying cursor, connection lifetime, exceptions, and transaction scope must remain valid until iteration completes.

## Pattern Selection and Failure

Choose based on the problem:

| Problem | Useful shape | Main risk |
| --- | --- | --- |
| Interchangeable rule | Strategy | Too many trivial classes |
| Queued or replayed operation | Command | Stale data and duplicate effects |
| Legal state transitions | State or explicit state machine | Check/write race |
| Independent reactions | Event/Observer | Delivery and schema drift |
| Ordered processing | Chain | Hidden order and short-circuit bugs |
| Fixed algorithm with narrow variation | Template Method | Inheritance coupling |
| Streaming traversal | Iterator/generator | Resource lifetime |

Failure behavior is part of the pattern. Define retries and idempotency for commands, transaction and outbox rules for events, authorization for every handler, and bounded work for chains and generators. A pattern that hides latency or side effects is a production risk.

## Testing Behavioral Patterns

Test each strategy against the shared contract and its boundary cases. Test commands for validation, authorization, duplicate handling, and retry recovery. Test state transitions with legal and illegal edges plus concurrent writes. Test observers for payload privacy, duplicate delivery, failure isolation, and schema versioning.

For chains, test handler order and short-circuit outcomes. For generators, test empty input, large input, exceptions, and resource cleanup. Use integration and feature tests for queues, transactions, serialization, and real middleware wiring; unit tests alone cannot prove delivery.

## Exercises

1. Replace a discount conditional tree with two Strategy implementations and a contract test.
2. Design a queued command for an order refund. Include authorization context, idempotency, schema version, and failure recovery.
3. Model a subscription state machine and test legal, illegal, and concurrent transitions.
4. Build an event subscriber for an outbox event. Decide which failures retry and which are dead-lettered.
5. Define a validation chain and document ordering, short-circuit, and fail-closed behavior.

## Review Questions

1. When is a Strategy clearer than a match expression?
2. What additional concerns appear when a Command becomes a queued message?
3. Why must state transitions be enforced atomically?
4. What coupling does an Observer remove, and what responsibilities does it add?
5. When is a chain harder to reason about than explicit code?
6. What resource risks must a generator or iterator document?

## Summary

Behavioral patterns structure variable algorithms, commands, state transitions, notifications, ordered handling, and traversal. Use Strategy for genuine variation, Commands for explicit and replayable operations, State for legal transitions, events for independently timed reactions, and chains or iterators when ordering or streaming is a real requirement. Define authorization, retries, idempotency, transaction, delivery, and resource semantics before adding the pattern.

## References

- [PHP Manual: Enumerations](https://www.php.net/manual/en/language.enumerations.php)
- [PHP Manual: Generators](https://www.php.net/manual/en/language.generators.php)
- [PHP Manual: Iterator](https://www.php.net/manual/en/class.iterator.php)
- [Refactoring.Guru: Behavioral Design Patterns](https://refactoring.guru/design-patterns/behavioral-patterns)
- [Martin Fowler: Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)

