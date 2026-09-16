---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 30
title: Interfaces
slug: interfaces
status: complete
summary: ../../_ai/chapter-summaries/030-interfaces-summary.md
---

# Chapter 30 — Interfaces

## Why This Matters

An interface names a capability that a class promises to provide. It lets a caller depend on behavior rather than a concrete implementation: a reservation service can publish an event to a database-backed publisher, a queue publisher, or a test fake.

That flexibility has a cost. Every interface is a public contract that must be understood, tested, and evolved. Adding an interface between two classes that never need independent variation can make a small system harder to navigate without improving its design.

## Mental Model

An interface is a type-level boundary. It describes callable methods and constants without supplying their implementations. Since PHP 8.4, an interface may also declare properties that implementing classes must satisfy, but the interface still does not provide the object’s storage or behavior. A class uses `implements` to promise the contract. A class may implement several interfaces; an interface may extend several interfaces.

The caller sees the stable capability. The implementation decides how to perform it. This is useful when the implementation is replaceable for a reason—different storage, an external provider, a fake in tests, or an infrastructure migration—not simply because abstraction is fashionable.

## Core Concept

```php
<?php

declare(strict_types=1);

interface EventPublisher
{
    public function publish(DomainEvent $event): void;
}

final class SynchronousEventPublisher implements EventPublisher
{
    public function __construct(private EventStore $store) {}

    public function publish(DomainEvent $event): void
    {
        $this->store->append($event);
    }
}

final class NullEventPublisher implements EventPublisher
{
    public function publish(DomainEvent $event): void {}
}
```

The empty implementation is appropriate only when “intentionally do nothing” is a real policy, not as a way to silence an error. The interface lets application code depend on `EventPublisher`:

```php
final class ReservationCreatedHandler
{
    public function __construct(private EventPublisher $publisher) {}

    public function handle(ReservationCreated $event): void
    {
        $this->publisher->publish($event);
    }
}
```

## How It Works

PHP checks interface implementation when the class is declared and checks the method call at runtime. A variable typed as an interface can hold any object that implements it. The engine then dispatches the call to the concrete method on the object’s class.

The interface is not a network boundary and does not provide isolation. If `publish()` calls a remote broker, it can still time out and throw. A good contract documents failure semantics through types, exceptions, or a result object, and through tests and operational documentation.

## What PHP Does

Interface contracts participate in type variance:

```php
interface Parser
{
    /** @return array<string, mixed> */
    public function parse(string $input): array;
}

final class JsonParser implements Parser
{
    public function parse(string $input): array
    {
        $value = json_decode($input, true, flags: JSON_THROW_ON_ERROR);

        if (!is_array($value)) {
            throw new UnexpectedValueException('JSON object expected');
        }

        return $value;
    }
}
```

The implementation must honor the declared return type. More generally, an implementation may accept a broader parameter type or return a more specific compatible type, but it must not make valid interface calls fail unexpectedly. PHP’s type compatibility is necessary; it is not a complete behavioral specification.

## Practical Example

Define the smallest useful port for a payment flow:

```php
interface PaymentGateway
{
    public function authorize(Money $amount, PaymentToken $token): Authorization;
}

final class Checkout
{
    public function __construct(private PaymentGateway $payments) {}

    public function pay(Money $amount, PaymentToken $token): Authorization
    {
        return $this->payments->authorize($amount, $token);
    }
}
```

The interface does not expose the gateway’s SDK request, HTTP client, or credentials. It is a port shaped around the application’s need. A production adapter translates that port to the provider; a test fake returns controlled authorizations or failures.

## Production Example

An interface is especially valuable at a boundary with different operational behavior:

```text
Application port: ReservationRepository
    ├── PDO adapter (transactional, indexed SQL)
    ├── read replica adapter (stale, read-only)
    └── in-memory fake (fast, test-only)
```

These implementations are not interchangeable in every situation. A read replica may be stale, and a fake does not prove SQL constraints. The contract should state what the caller can rely on, while integration tests cover adapter-specific behavior.

## Bad Example

```php
interface HasId
{
    public function getId(): string;
}

interface HasName {}
interface HasCreatedAt {}
interface CanBeConvertedToArray {}
```

Marker and getter interfaces often describe incidental structure rather than a useful capability. They multiply types without giving a caller a meaningful guarantee. An interface should answer: what operation does a client need, and what behavior does it receive?

## Better Example

Expose a behavior with its failure contract:

```php
interface ReservationAvailability
{
    /**
     * Returns false for a confirmed conflict; throws when availability cannot be determined.
     */
    public function isAvailable(CourtId $court, TimeRange $range): bool;
}
```

Now a caller can distinguish a normal business answer from an infrastructure failure. If the operation needs more states—available, occupied, unknown, or blocked—an enum or result object may be clearer than overloaded exceptions and booleans.

## Edge Cases

- Implementing a method with a narrower parameter type can make valid interface callers fail and is rejected by PHP’s signature rules.
- Interface constants are public API. Changing their meaning can break clients even when the code still compiles.
- An interface can extend multiple interfaces, but a wide inherited contract may become difficult to implement.
- Since PHP 8.4, interface properties are part of the contract; account for that when supporting older PHP versions or when using property hooks.
- A class can be both an interface implementation and a child of another class.
- Keep an interface focused, but do not split methods into artificial fragments merely to achieve small numbers.
- An interface does not force implementations to be stateless, deterministic, fast, or idempotent; document those requirements.

## Performance

Interface dispatch is generally a small in-process cost and should not drive architecture. The expensive difference is usually the implementation behind the interface: a SQL query, HTTP call, serialization, or queue operation. Do not add an interface to a hot loop as a performance technique; measure first.

## Security

An interface is not an authorization boundary. Any implementation supplied to a caller can implement the methods while violating security expectations unless the caller controls construction and the contract is enforced. Keep authorization close to the protected resource, pass scoped capability objects, and test denial paths against real adapters where appropriate.

## Database Interaction

Repository interfaces should not erase important database semantics. If `save()` can fail on a unique constraint, the contract should say what the application receives. If reads are eventually consistent, model that choice rather than pretending a replica is a primary database. Keep SQL and query-plan tests in adapter tests; keep domain behavior in port-level tests.

## Concurrency

An interface does not serialize concurrent calls. Two PHP workers can call the same implementation at once, and a remote provider can deliver duplicate callbacks. State-changing methods should define idempotency, transaction boundaries, and retry behavior. A fake that simply appends to an array cannot prove those properties.

## Testing

Test the consumer against a behavioral fake:

```php
final class FakePaymentGateway implements PaymentGateway
{
    /** @var list<Money> */
    public array $authorized = [];

    public function authorize(Money $amount, PaymentToken $token): Authorization
    {
        $this->authorized[] = $amount;

        return Authorization::approved('test-auth');
    }
}

final class CheckoutTest extends TestCase
{
    public function testCheckoutUsesThePaymentPort(): void
    {
        $gateway = new FakePaymentGateway();
        $checkout = new Checkout($gateway);

        $checkout->pay(Money::eur(100), PaymentToken::fake());

        self::assertCount(1, $gateway->authorized);
    }
}
```

Add contract tests shared by every production adapter when multiple implementations promise the same behavior. Then add adapter integration tests for SQL, HTTP error mapping, timeouts, and provider-specific constraints.

## Common Mistakes

- Creating interfaces for every class before a second implementation or meaningful boundary exists.
- Naming an interface after an implementation detail such as `PdoReservationRepositoryInterface`.
- Omitting latency, failure, consistency, or idempotency from a boundary contract.
- Testing only that a mock method was called rather than testing the resulting behavior.
- Treating a fake as proof that a database adapter is correct.

## Senior Engineer Thinking

Design an interface from the client’s need, not from the methods a concrete class happens to have. Ask who owns the contract, how it evolves, and whether an adapter really can honor it. If two implementations differ in critical guarantees, use separate interfaces or make those guarantees explicit instead of hiding the difference.

## Exercises

1. Extract the smallest interface needed by a reservation service to publish a confirmation.
2. Add a contract test that both an in-memory and PDO repository must pass.
3. Identify a boolean-returning interface method whose states are really three or more outcomes; redesign it with an enum or result object.
4. Decide whether a clock interface is justified in a small script and in a long-lived booking service. Explain the constraints that change your answer.

## Review Questions

1. What does an interface promise, and what does it leave unspecified?
2. Why is an interface not automatically a reliability or security boundary?
3. When does an interface reduce coupling?
4. Why do fakes and adapter integration tests provide different confidence?
5. What should a repository contract say about unique conflicts and stale reads?

## Summary

Interfaces define replaceable capabilities at meaningful boundaries. Keep them client-shaped and behaviorally honest, use variance-compatible signatures, and document failure, consistency, and idempotency. Use fakes for consumer tests and real adapter tests for database and network behavior; do not abstract every class by reflex.
