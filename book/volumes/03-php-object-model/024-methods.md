---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 24
title: Methods
slug: methods
status: complete
summary: ../../_ai/chapter-summaries/024-methods-summary.md
---

# Chapter 24 — Methods

## Why This Matters

A property tells you what an object has; a method tells you what it can do. Good methods turn a requirement into a named operation with a clear input, output, invariant, and side-effect policy. Poor methods are vague bags of commands, return inconsistent types, mutate surprising collaborators, or hide a query behind a name that sounds harmless.

Methods are where object design becomes behavior. The goal is not to maximize encapsulation by making every operation indirect. The goal is to make important decisions observable in the API and difficult to misuse.

## Mental Model

An instance method runs with a current object available as `$this`:

```php
final class Temperature
{
    public function __construct(private float $celsius)
    {
    }

    public function fahrenheit(): float
    {
        return ($this->celsius * 9 / 5) + 32;
    }
}
```

`$temperature->fahrenheit()` invokes the method on one object. The method can read and mutate that object’s properties, call other methods, and return a value. A method’s name should describe its observable behavior. `confirm()` implies a state transition; `status()` implies observation; `send()` implies an external side effect.

## Core Concept

A method contract has at least four parts:

- parameters and their accepted types;
- the return type and meaning of each result;
- state changes and exceptions;
- external side effects, timing, and retry expectations.

```php
final class Reservation
{
    public function __construct(private string $status = 'pending')
    {
    }

    public function confirm(): void
    {
        if ($this->status !== 'pending') {
            throw new DomainException('Only pending reservations can be confirmed.');
        }

        $this->status = 'confirmed';
    }

    public function isConfirmed(): bool
    {
        return $this->status === 'confirmed';
    }
}
```

`confirm()` has a narrow command contract: it either changes pending state to confirmed or throws. `isConfirmed()` is a query and should not change state. This command/query distinction is a useful design heuristic even when a system does not implement full CQRS.

## How It Works

PHP resolves a method call from the object’s class and visibility rules, prepares the arguments, enters the method body, and checks the declared return type as the method returns. A method can be overloaded by inheritance in later chapters, but PHP does not support traditional signature-based method overloading within one class. Use optional parameters, named constructors, or distinct method names when operations have different inputs.

Parameters are local variables. An object parameter receives a copy of the object handle, so a method can mutate the object it receives, but assigning a new object to the parameter does not replace the caller’s variable. Return an object when the caller should continue with the same or a new instance; return a scalar or value object when that is the actual result.

A `void` method may use `return;` but cannot return a value. `never` is appropriate for a method that always throws or terminates, but it should describe a real control-flow guarantee, not be used to avoid designing a result. A `static` method has no `$this`; use it for behavior belonging to the class rather than one instance, such as a named factory or pure utility that has no object state.

## What PHP Does

Method calls are runtime operations. The method body may perform calculations, mutate properties, throw exceptions, query a database, or call another process. PHP’s type declarations catch many boundary errors, but they cannot determine whether a method’s side effect is safe to repeat.

A method that sends an email, charges a card, or writes a row is not equivalent to a pure method that formats a string. Give side effects a visible name and dependency. If a request can retry, design an idempotency key or durable state transition rather than hoping the method is called once.

## Minimal Example

An object can expose a small, intention-revealing API:

```php
final class ShoppingCart
{
    /** @var array<string, int> */
    private array $quantities = [];

    public function add(string $sku, int $quantity): void
    {
        if ($sku === '' || $quantity < 1) {
            throw new InvalidArgumentException('Sku and quantity are invalid.');
        }

        $this->quantities[$sku] = ($this->quantities[$sku] ?? 0) + $quantity;
    }

    public function quantityFor(string $sku): int
    {
        return $this->quantities[$sku] ?? 0;
    }

    /** @return array<string, int> */
    public function quantities(): array
    {
        return $this->quantities;
    }
}
```

`add()` names a business operation. `quantityFor()` is a query. `quantities()` returns an array copy-on-write value, but the method still chooses whether exposing the full representation is appropriate. If the array contains mutable objects, returning it would expose shared children.

## Practical Example

An interval method can express a rule without exposing its arithmetic to every caller:

```php
final class TimeSlot
{
    public function __construct(
        private DateTimeImmutable $startsAt,
        private DateTimeImmutable $endsAt,
    ) {
        if ($endsAt <= $startsAt) {
            throw new InvalidArgumentException('End must be after start.');
        }
    }

    public function overlaps(self $other): bool
    {
        return $this->startsAt < $other->endsAt()
            && $other->startsAt() < $this->endsAt;
    }

    public function startsAt(): DateTimeImmutable
    {
        return $this->startsAt;
    }

    public function endsAt(): DateTimeImmutable
    {
        return $this->endsAt;
    }
}
```

For half-open intervals, `[start, end)` and `[end, later)` do not overlap. The method centralizes the rule so the HTTP layer, a command handler, and a database adapter do not independently invent different comparisons. The database must still enforce the final concurrency invariant.

## Production Example

Methods often coordinate dependencies:

```php
interface Clock
{
    public function now(): DateTimeImmutable;
}

final class ConfirmReservation
{
    public function __construct(private Clock $clock)
    {
    }

    public function handle(Reservation $reservation): DateTimeImmutable
    {
        $at = $this->clock->now();
        $reservation->confirm();

        return $at;
    }
}
```

Injecting a clock makes time an explicit dependency and removes the need for tests to race the system clock. In a complete application, the caller would persist the state transition in a transaction and record the timestamp consistently with the domain rule. The method’s return value should mean something: here it is the chosen confirmation time, not an arbitrary copy of the object.

## Bad Example

```php
final class ReservationService
{
    public function process(array $data): mixed
    {
        // Sometimes returns false, sometimes an array, sometimes throws.
        // Validates input, queries three tables, charges a card, and emails.
    }
}
```

The name and return type conceal the contract. Callers cannot know which failures are expected, what has been committed when an exception occurs, or whether retrying is safe.

## Better Example

Split the behavior around stable decisions and make outcomes explicit:

```php
final class ReservationResult
{
    private function __construct(
        public readonly string $reservationId,
        public readonly string $status,
    ) {
    }

    public static function confirmed(string $id): self
    {
        return new self($id, 'confirmed');
    }
}

final class ReservationApplication
{
    public function __construct(private ReservationRepository $repository)
    {
    }

    public function reserve(ReservationRequest $request): ReservationResult
    {
        $id = $this->repository->insertIfAvailable($request);

        return ReservationResult::confirmed($id);
    }
}
```

The repository method name states an important requirement: the availability check and insert must have an atomic implementation. If it cannot guarantee that, the method should not pretend it does. A result object can later grow a pending or rejected outcome without making callers interpret `false` and `null` differently.

## Static Methods and Named Constructors

Static methods can provide readable creation paths:

```php
final class EmailAddress
{
    private function __construct(private string $value)
    {
    }

    public static function fromString(string $value): self
    {
        $normalized = strtolower(trim($value));

        if (filter_var($normalized, FILTER_VALIDATE_EMAIL) === false) {
            throw new InvalidArgumentException('Invalid email address.');
        }

        return new self($normalized);
    }

    public function value(): string
    {
        return $this->value;
    }
}
```

The private constructor forces callers through validation. This is useful when there are multiple representations or the public creation language matters. Do not turn every one-line `new` into a factory; the extra name is justified when it communicates parsing, normalization, or a distinct invariant.

## Edge Cases

- Calling an instance method statically is an error in modern PHP; a method that needs no instance state should be declared static or moved to a function/class with a clearer role.
- A method can be variadic, accept named arguments, and use union/intersection types, but a wide signature can be a sign that the input deserves a value object.
- Fluent methods returning `$this` share the same mutable instance. Returning a new object is a different contract; document the distinction.
- A method may call another method that is overridden in a child class. Inheritance can therefore change behavior indirectly; use `final` or composition when substitution is not intended.
- Exceptions are part of a method’s practical contract even though PHP does not declare them in the signature. Document expected domain failures and keep infrastructure failures distinguishable.

## Performance

A method call itself is rarely the dominant cost in a PHP application. Database round trips, network latency, serialization, and allocation usually matter more. Avoid premature inlining or static utility tricks that make the boundary less testable.

Do pay attention to accidental complexity inside methods. A `hasOverlap()` method that scans every reservation is O(n); an indexed database query or sorted interval structure may fit the constraints better. A getter that loads a relation for every row creates an N+1 query pattern. Name methods so expensive work is visible, or expose an explicit loading operation.

## Security

A public method is an API surface. Validate input at the boundary, authorize the operation using the authenticated actor and resource scope, and do not trust a method name as proof that a caller is allowed to invoke it. Keep dangerous operations narrow: `deleteForUser(UserId $actor, ReservationId $id)` carries more context than `delete(string $id)`.

For side-effecting methods, use parameterized database queries and safe output encoding in their collaborators. Do not place secrets in parameters that are routinely logged, and do not include raw request data in exception messages.

## Testing

Pure queries and state transitions are easy to test directly:

```php
function test_cart_add_accumulates_quantity(): void
{
    $cart = new ShoppingCart();
    $cart->add('ball', 2);
    $cart->add('ball', 3);

    assert($cart->quantityFor('ball') === 5);
}

function test_confirmation_uses_the_injected_clock(): void
{
    $expected = new DateTimeImmutable('2026-01-01 10:00:00 UTC');
    $clock = new class($expected) implements Clock {
        public function __construct(private DateTimeImmutable $time) {}
        public function now(): DateTimeImmutable { return $this->time; }
    };

    $reservation = new Reservation();
    $result = new ConfirmReservation($clock)->handle($reservation);

    assert($result == $expected);
}
```

For integration tests, verify transaction boundaries, duplicate requests, and the database’s conflict response. A unit test that mocks every collaborator can prove only that calls occurred; it cannot prove that the system remains correct when a commit or network call fails.

## Common Mistakes

- Naming a method after an implementation detail instead of an intention.
- Returning `mixed`, `false`, `null`, and arrays for unrelated outcomes.
- Hiding a query, network call, or write behind a getter or predicate.
- Calling a mutable fluent method as if it returned a new value.
- Depending directly on the current time, random generator, or global state.
- Assuming a pre-check method makes a concurrent write safe.

## Senior Engineer Thinking

Read a method signature as a miniature design document. Can a caller tell what it needs, what it gets, what may change, and what happens on retry? If not, improve the contract before adding logging or tests around ambiguous behavior.

Prefer a small number of operations that protect meaningful invariants over a large collection of getters and setters. Put pure calculations close to the data they interpret, but keep orchestration and external side effects visible at application boundaries.

## Exercises

1. Add `cancel()` to `Reservation` with a legal-transition table for pending, confirmed, and cancelled states.
2. Refactor a method returning `mixed` into a result object and typed exceptions. List each caller that becomes simpler.
3. Implement `TimeSlot::overlaps()` tests for touching, contained, identical, and disjoint intervals.
4. Replace direct `new DateTimeImmutable()` calls in a service with an injected clock and test a boundary at exactly midnight.

## Review Questions

1. What belongs in a method contract besides its parameter and return types?
2. Why should a query method avoid hidden mutation or I/O?
3. When is a static named constructor useful, and when is it unnecessary ceremony?
4. Why does an availability pre-check not guarantee a safe concurrent reservation?
5. What does injecting a clock improve in production code and tests?

## Summary

Methods are behavioral contracts. Name them by intention, type their inputs and outputs, make state transitions explicit, and expose external side effects and retry behavior. Keep pure queries separate from commands where that clarifies the design, inject unstable dependencies such as time, and let the database enforce concurrent invariants. Visibility determines who may call these methods, which is the focus of the next chapter.
