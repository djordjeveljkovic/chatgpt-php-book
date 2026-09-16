---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 23
title: Properties
slug: properties
status: complete
summary: ../../_ai/chapter-summaries/023-properties-summary.md
---

# Chapter 23 — Properties

## Why This Matters

Properties are the state an object carries between method calls. They look like variables, but their declaration communicates more: type, visibility, whether initialization is required, whether `null` is meaningful, and whether callers may write the value. A property design is therefore a data-model decision and an API decision at the same time.

An untyped public property can accept almost anything from anywhere. A typed private property can make invalid states harder to represent. Neither choice is automatically correct; the right choice follows from ownership and invariants.

## Mental Model

A non-static property belongs to an object instance. A static property belongs to the class and is shared across instances. A declaration can include a type, a visibility modifier, and a default value:

```php
final class Match
{
    private int $games = 0;
    private ?string $winner = null;
}
```

`$games` starts at zero. `$winner` starts as `null`, which explicitly represents “not decided.” A typed property without a default has a third state: declared but uninitialized. Accessing it before assignment throws an `Error`; it is not the same as `null`.

## Core Concept

Use types to describe the allowed state, then choose where writes occur:

```php
final class ReservationState
{
    private string $status;

    public function __construct()
    {
        $this->status = 'pending';
    }

    public function status(): string
    {
        return $this->status;
    }

    public function confirm(): void
    {
        if ($this->status !== 'pending') {
            throw new DomainException('Only pending reservations can be confirmed.');
        }

        $this->status = 'confirmed';
    }
}
```

The property is private because the class owns the state transition. The public API says “confirm this reservation,” not “write any string into status.” Visibility is explored deeply in Chapter 25; the important property lesson is that storage and public meaning are not the same thing.

## How It Works

Typed properties were introduced in PHP 7.4. They are checked when written and when read. A property declared as `int` rejects a string under the relevant strictness rules; a property declared `?int` accepts an integer or `null`.

```php
final class User
{
    public int $id;
    public ?string $nickname = null;
}

$user = new User();
$user->id = 42;
$user->nickname = null;
assert($user->id === 42);
```

`$user->id` cannot be read before the assignment. The constructor is a natural place to initialize required state, but a method can initialize a property later when the lifecycle genuinely has phases. If delayed initialization is valid, document the state machine; do not leave it accidental.

Property defaults must be constant expressions. This is invalid because the current time is runtime state:

```php
// private DateTimeImmutable $createdAt = new DateTimeImmutable(); // invalid property default
```

Use a constructor for a timestamp. Modern PHP permits `new` in several initializer contexts, such as parameter defaults and static variables, but that does not make a runtime clock value a sensible property default. Prefer explicit construction for domain timestamps.

## What PHP Does

For an instance property, `$this->name` resolves the property on the current object. For a static property, `self::$count` or `ClassName::$count` resolves class state. Static state survives across calls in the same process and can therefore surprise code running in workers or tests. It is not a substitute for a database or cache with a defined consistency policy.

PHP 8.2 deprecated creating undeclared dynamic properties in ordinary classes. Code like `$user->nmae = 'typo';` can silently create the wrong field in older code; modern code should declare the property or intentionally use a supported dynamic design. `#[AllowDynamicProperties]` is a migration escape hatch, not a reason to keep unbounded object shape in new domain code.

PHP 8.1 introduced `readonly` properties, which can be initialized once and not modified afterward. PHP 8.4 added property hooks and asymmetric property visibility, allowing read and write behavior to be expressed at the property boundary. These features are useful, but they do not remove the need to choose an invariant and an owner. The later readonly chapter goes deeper; this chapter focuses on ordinary properties.

## Minimal Example

Model a coordinate with typed state:

```php
final class Point
{
    public function __construct(
        private int $x,
        private int $y,
    ) {
    }

    public function x(): int
    {
        return $this->x;
    }

    public function y(): int
    {
        return $this->y;
    }
}
```

The properties are private, so callers cannot put the object into a half-valid shape by writing arbitrary values. The accessors are intentionally boring. Boring is good when reading a value has no extra policy.

## Practical Example

An interval has stronger property requirements than a pair of unconstrained timestamps:

```php
final class TimeSlot
{
    public function __construct(
        private DateTimeImmutable $startsAt,
        private DateTimeImmutable $endsAt,
    ) {
        if ($endsAt <= $startsAt) {
            throw new InvalidArgumentException('A time slot must have positive length.');
        }
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

`DateTimeImmutable` is a useful property type because callers cannot mutate the stored timestamp through the returned object. If the property were a mutable `DateTime`, returning it would give callers another path to change internal state. A property type is therefore also a choice about aliasing.

## Production Example

A persisted entity often has fields with different lifecycle stages:

```php
final class ReservationRecord
{
    private ?string $databaseId = null;
    private string $status = 'pending';
    private ?DateTimeImmutable $confirmedAt = null;

    public function assignDatabaseId(string $id): void
    {
        if ($this->databaseId !== null) {
            throw new LogicException('Database id is already assigned.');
        }

        if ($id === '') {
            throw new InvalidArgumentException('Database id cannot be empty.');
        }

        $this->databaseId = $id;
    }

    public function confirm(DateTimeImmutable $at): void
    {
        if ($this->status !== 'pending') {
            throw new DomainException('Reservation is not pending.');
        }

        $this->status = 'confirmed';
        $this->confirmedAt = $at;
    }
}
```

The nullable properties are not “weak typing”; they model a lifecycle where a database id or confirmation timestamp does not exist yet. The methods enforce one-way transitions. A mapper may need controlled access to reconstitute a record, but bypassing all methods with public writes makes the invariant dependent on every caller remembering the same rules.

## Bad Example

```php
final class Reservation
{
    public $courtId;
    public $startsAt;
    public $endsAt;
    public $status;
}

$reservation->status = 'confirmed';
$reservation->endsAt = 'yesterday';
```

This code has no declared types, no initialization contract, no interval rule, and no controlled state transition. A later validation function may help, but every path that obtains the object must remember to call it.

## Better Example

Use a typed boundary and methods that express intent:

```php
final class Reservation
{
    public function __construct(
        private int $courtId,
        private TimeSlot $slot,
        private string $status = 'pending',
    ) {
        if ($courtId < 1 || !in_array($status, ['pending', 'confirmed'], true)) {
            throw new InvalidArgumentException('Invalid reservation state.');
        }
    }

    public function confirm(): void
    {
        if ($this->status !== 'pending') {
            throw new DomainException('Reservation cannot be confirmed twice.');
        }

        $this->status = 'confirmed';
    }
}
```

This is not a complete entity—there are no getters yet because the example is about writes—but it shows the direction: properties are selected around ownership, not around the columns that happen to exist today.

## Edge Cases

- `?T` means `T|null`; it does not mean “possibly uninitialized.” An uninitialized typed property must be initialized before reading.
- `isset($object->property)` returns false for `null` and for an inaccessible/uninitialized property in contexts where it cannot be read. `property_exists()` checks declaration existence and may return true for a null property.
- `public`, `protected`, and `private` can apply to properties. An omitted visibility historically means public; prefer explicit modifiers in new code.
- `callable` is not a permitted property type because of ambiguity in how callable values would be resolved; use an appropriate closure/interface design instead.
- A property declared `readonly` cannot be treated as a mutable collection just because the collection object itself is mutable. Readonly protects the property slot, not every nested object.
- PHP 8.4 property hooks can intercept reads and writes, but they should not be used to conceal expensive I/O behind ordinary property syntax.

## Performance

Typed properties and sensible object boundaries usually improve correctness more than they affect runtime. Performance problems arise from object graph size, repeated hydration, retained references, and hidden work in accessors/hooks. For large result sets, stream or project only the fields needed rather than creating a full graph for every row.

Avoid copying large arrays into properties unnecessarily. PHP’s copy-on-write behavior can make scalar/array assignment cheap until mutation, but that optimization is an implementation detail you should not use to justify unclear ownership. Measure memory in the actual PHP process model, especially for workers.

## Security

Properties holding secrets, tokens, passwords, or personal data should have narrow visibility and controlled logging. A private property is not encryption: reflection, debugging tools, serialization, or a compromised process may still expose it. Store passwords as hashes using the password API and avoid retaining plaintext longer than needed.

Do not hydrate arbitrary property names from request input. Map an allow-listed input shape to explicit properties and validate both type and business meaning. Mass assignment is a boundary problem, not a convenience feature to trust blindly.

## Testing

Test initialization and invariants through public behavior:

```php
function test_invalid_slot_is_rejected(): void
{
    $time = new DateTimeImmutable('2026-01-01 10:00:00 UTC');

    try {
        new TimeSlot($time, $time);
        throw new RuntimeException('Expected InvalidArgumentException.');
    } catch (InvalidArgumentException) {
        // Expected.
    }
}

function test_confirmation_sets_a_timestamp(): void
{
    $at = new DateTimeImmutable('2026-01-01 10:00:00 UTC');
    $record = new ReservationRecord();

    $record->confirm($at);
    // Assert status and confirmedAt through deliberate read methods in the real class.
}
```

Do not use reflection in ordinary unit tests just to inspect private fields. If a state matters, expose a meaningful query such as `status()` or `confirmedAt()`. Test database mappers separately for null handling, missing columns, and schema/type conversion.

## Common Mistakes

- Using public properties as a default API for mutable domain state.
- Confusing `null` with an uninitialized typed property.
- Returning mutable child objects and calling the parent immutable.
- Keeping static mutable state in code that runs in workers or parallel tests.
- Using `#[AllowDynamicProperties]` to hide typos in new code.
- Putting database queries or network calls in property accessors or hooks.

## Senior Engineer Thinking

For each property, write four answers: what values are valid, who initializes it, who may change it, and how long the value is valid. If you cannot answer those questions, the property is probably carrying an unmodeled state transition. If a property is public, state why external code is allowed to write it and how invalid combinations are prevented.

## Exercises

1. Add read methods to `ReservationRecord` and test its pending-to-confirmed transition, including the second-confirmation failure.
2. Rewrite a class with untyped public properties using typed private properties. List each newly explicit invariant.
3. Create a `Money` class with integer minor units and a three-letter currency. Decide whether both properties should be readonly and explain why.
4. Find a dynamic-property warning in a legacy code sample and migrate it to declared properties or an intentional `__get`/`__set` design. Do not suppress the warning without documenting the boundary.

## Review Questions

1. What is the difference between a nullable property and an uninitialized typed property?
2. Why does returning a mutable object-valued property weaken encapsulation?
3. Which lifecycle facts justify a nullable database id or timestamp?
4. Why are public properties often a poor default for domain state?
5. What did PHP 8.2 change about undeclared dynamic properties, and what is the safer new-code direction?

## Summary

Properties are typed, owned state—not merely columns or public variables. Choose their type, initialization, nullability, visibility, and mutability from the object’s invariants and lifecycle. Typed properties catch invalid assignments and make uninitialized state visible; immutable child values reduce aliasing. Declare properties explicitly, keep writes intentional, and keep I/O out of property access so the next chapter can give methods a clear behavioral contract.
