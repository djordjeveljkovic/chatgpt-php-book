---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 34
title: Readonly
slug: readonly
status: complete
summary: ../../_ai/chapter-summaries/034-readonly-summary.md
---

# Chapter 34 — Readonly

## Why This Matters

Many objects are created as facts: an amount has been validated, a reservation has an identifier, and a command has captured its input. If those facts can be silently replaced later, the reader must reason about every possible mutation. `readonly` narrows that state space.

Readonly is not a synonym for “deeply immutable.” It controls reassignment of a property or the declared properties of a class. An object stored inside a readonly property can still mutate internally. Understanding that boundary prevents both underestimating the feature and promising more than it provides.

## Mental Model

```text
readonly property: the slot may be initialized once
object in the slot: that object's own API still decides mutability
```

Readonly properties arrived in PHP 8.1. A readonly class arrived in PHP 8.2. In PHP 8.4, a readonly property is implicitly `protected(set)` rather than implicitly private-set, so a child may initialize an inherited property unless the declaration chooses stricter set visibility. Check the project's minimum PHP version before using these features.

## Core Concept

```php
<?php

declare(strict_types=1);

final class Reservation
{
    public function __construct(
        public readonly string $id,
        public readonly DateTimeImmutable $startsAt,
        public readonly DateTimeImmutable $endsAt,
    ) {
        if ($endsAt <= $startsAt) {
            throw new InvalidArgumentException('End must follow start.');
        }
    }
}
```

The constructor is the write boundary. Reassignment, increment, array-element modification, `unset`, and by-reference modification of an initialized readonly property all fail with `Error`. A readonly property must be typed; `mixed` is the explicit unrestrictive type. Static properties cannot be readonly.

## How It Works

Initialization must be a direct assignment from the declaring scope. A promoted readonly property is initialized during construction. A readonly property with no default can be initialized later by a method in the declaring class, but only once. A default value is forbidden because a property with a default would already be initialized and would behave like a constant.

Readonly arrays cannot be changed through the property:

```php
final class Labels
{
    public function __construct(public readonly array $values)
    {
    }
}

$labels = new Labels(['new']);
// $labels->values[] = 'paid'; // Error: cannot modify readonly property
```

Objects are different:

```php
final class Basket
{
    public function __construct(public readonly ArrayObject $items)
    {
    }
}

$basket = new Basket(new ArrayObject());
$basket->items->append('reservation'); // allowed: interior mutation
// $basket->items = new ArrayObject(); // forbidden: slot reassignment
```

This is shallow immutability. Use immutable collaborators such as `DateTimeImmutable`, defensive copies, or value objects when the graph itself must not change.

## Readonly Classes

```php
readonly class Money
{
    public function __construct(
        public int $minorUnits,
        public string $currency,
    ) {
        if ($minorUnits < 0 || !preg_match('/^[A-Z]{3}$/', $currency)) {
            throw new InvalidArgumentException('Invalid money value.');
        }
    }
}
```

A readonly class makes every declared instance property readonly and forbids dynamic properties. It cannot declare untyped or static properties, and a child must also be readonly. `#[AllowDynamicProperties]` cannot opt out. A readonly class may still have methods that perform external side effects; readonly describes object state, not the purity of its methods.

## Practical Example: Commands and State Transitions

Readonly is well suited to an input command:

```php
final readonly class CreateReservation
{
    public function __construct(
        public string $courtId,
        public DateTimeImmutable $startsAt,
        public DateTimeImmutable $endsAt,
    ) {
        if ($endsAt <= $startsAt) {
            throw new InvalidArgumentException('Invalid interval.');
        }
    }
}
```

The handler can trust that the command it received will not be rewritten by another helper. It must still validate authorization and database state. A readonly request object does not make an untrusted HTTP payload trustworthy; validation occurs before construction.

## Bad Example and Better Example

Bad code stores a mutable service in a readonly property and calls that “immutable”:

```php
final class BadCacheHolder
{
    public function __construct(public readonly ArrayObject $cache) {}
}
```

Better code states the intended boundary:

```php
final readonly class ReservationSnapshot
{
    /** @param list<string> $reservationIds */
    public function __construct(public array $reservationIds) {}
}
```

If a snapshot contains objects, define whether it owns copies, references immutable values, or is only a stable container. Naming and documentation are part of the contract.

## Edge Cases and Cloning

PHP 8.3 allows a readonly property to be reinitialized during `__clone()` on the newly cloned object. This supports copy-with-changes patterns, but the assignment must be direct and legal for the clone operation. PHP 8.4 tightened indirect modification inside `__clone()`, so do not acquire references to readonly properties as a workaround.

Readonly does not prevent serialization, reflection by privileged code, or mutation of interior objects. It also does not make a class thread-safe or make its external dependencies reliable.

## Performance

Readonly can simplify reasoning and reduce accidental copying of defensive state, but it is not automatically faster. Arrays remain arrays, objects remain object graphs, and constructing a replacement value still allocates. Measure allocation and serialization costs for large graphs rather than assuming a modifier changes their complexity.

## Security and Operations

Readonly DTOs reduce accidental request-state mutation and make audit logs more trustworthy within a process. They do not validate input, encrypt secrets, or prevent an attacker from submitting a different payload. Never put credentials in a broadly logged object merely because its properties are readonly. In workers, readonly state also does not reset process-global mutable collaborators; lifecycle boundaries still matter.

## Testing

Test both the intended write contract and interior mutability policy:

```php
$reservation = new Reservation(
    'r-1',
    new DateTimeImmutable('2026-09-14 10:00 UTC'),
    new DateTimeImmutable('2026-09-14 11:00 UTC'),
);

assert($reservation->id === 'r-1');

try {
    $reservation->id = 'r-2';
    assert(false, 'Expected readonly failure.');
} catch (Error $error) {
    assert(str_contains($error->getMessage(), 'readonly'));
}
```

Prefer PHPUnit assertions in production tests. Test construction invariants, serialization boundaries, clone behavior, and any nested object policy. Run a version matrix if your package supports PHP 8.1 through newer PHP 8.x releases.

## Common Mistakes

- Treating readonly as deep immutability.
- Trying to use readonly on static or untyped properties.
- Mutating a readonly array through an alias or reference.
- Assuming readonly removes the need for input validation.
- Depending on PHP 8.3 clone behavior while supporting PHP 8.2.

## Senior Engineer Thinking

Choose readonly when the lifecycle is “construct, observe, replace with a new value.” Do not use it to disguise a mutable aggregate or a service locator. For a value object, pair readonly properties with validation and immutable nested values. For an entity whose state legitimately transitions, private setters or explicit transition methods may describe the domain better.

## Exercises

1. Build a readonly `TimeInterval` and test its ordering invariant.
2. Put an `ArrayObject` in a readonly property, demonstrate interior mutation, and redesign it using an immutable representation.
3. Compare a readonly command with a mutable request array. Identify which bugs disappear and which security checks remain necessary.

## Review Questions

1. What exactly does a readonly property protect?
2. Why can an object inside a readonly property still mutate?
3. What restrictions apply to readonly classes?
4. Why is readonly not input validation?
5. When is an explicit state transition preferable to readonly?

## Summary

Readonly closes a property's reassignment lifecycle, not an entire object graph. Use it for validated facts, commands, snapshots, and value objects when replacement is clearer than mutation. Pair it with immutable nested values, validation, tests, and a version policy.

## Official References

- [PHP Manual: Properties and readonly](https://www.php.net/manual/en/language.oop5.properties.php)
- [PHP Manual: The Basics and readonly classes](https://www.php.net/manual/en/language.oop5.basic.php)
