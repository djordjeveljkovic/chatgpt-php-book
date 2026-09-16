---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 21
title: Classes
slug: classes
status: complete
summary: ../../_ai/chapter-summaries/021-classes-summary.md
---

# Chapter 21 — Classes

## Why This Matters

A class is not a fashionable way to wrap a function. It is a named boundary for a concept: the data that concept owns, the operations it supports, and the rules that must remain true. When a reservation system has a `Reservation` class, the useful question is not “how do I make an object?” It is “which facts and decisions belong together, and which code should be allowed to make them?”

Classes help us make those boundaries visible. They give PHP a type that can be checked by the language, documented for readers, and used as a seam in tests. They can also make a system worse when they merely rename arrays, hide simple calculations, or accumulate unrelated responsibilities. Object-oriented PHP is a tool for modeling constraints, not a requirement that every line be inside a class.

## Mental Model

Think of a class as a definition and an object as one runtime value created from that definition. The class describes a shape and behavior; an object carries the state for one identity. A class may declare properties, methods, constants, and a constructor. The declarations are shared by all instances, while each instance has its own property values.

The class is also a type. A function can require `Reservation` rather than an unstructured array, which moves part of the contract into the language:

```php
declare(strict_types=1);

function confirm(Reservation $reservation): void
{
    // The caller must provide an instance of Reservation.
}
```

This type check does not prove that the reservation is valid. It proves only that the value has the expected class identity. Invariants still need to be designed and enforced.

## Core Concept

A minimal class has a declaration and a method:

```php
final class Court
{
    public function __construct(
        public readonly int $id,
        public readonly string $name,
    ) {
    }

    public function label(): string
    {
        return "Court {$this->id}: {$this->name}";
    }
}

$court = new Court(3, 'Centre court');
echo $court->label();
```

`Court` is the class name. `new Court(...)` creates an object. The `$this` pseudo-variable means “the object on which this method was called.” Constructor property promotion and `readonly` are covered in detail later; they are used here to keep the first example small.

Class names are case-insensitive in PHP, but relying on unusual casing is poor practice. Use a stable, StudlyCaps name, one class per conceptual responsibility, and a namespace in application code:

```php
namespace App\Reservation;

final class Court
{
}
```

The namespace becomes part of the class’s fully qualified name. `App\Reservation\Court` and `Infrastructure\Reservation\Court` are different types even if their short names match. `use` imports a name for readability; it does not copy or instantiate a class.

## How It Works

When PHP reads a class declaration, it registers the class definition in the current execution context. The definition contains metadata for its methods, properties, constants, visibility, and type information. A `new` expression asks the runtime to create an instance of that class and then initializes it, including calling `__construct()` when one exists.

The class declaration does not execute every method body. Defining a method makes it available; calling the method executes its body. This distinction matters for side effects. A class file may be safely loaded without opening a database connection if construction and method calls are the points where I/O happen.

In a normal web request, the class definition and its objects live within the request’s process context. OPcache may cache compiled script data, but that does not make ordinary object state global across requests. A PHP-FPM worker can serve many requests, yet request-owned objects should not be treated as durable storage. See the execution and lifecycle discussion in Volume I and the later runtime chapters.

## What PHP Does

PHP checks class names, syntax, visibility, and declared types as code is compiled and executed. It raises an `Error` when a class cannot be found or an inaccessible member is used, and a `TypeError` when a value violates a parameter or return type. These are language failures, not validation results you should silently ignore.

Autoloading changes how a class definition becomes available. With Composer or another registered autoloader, PHP can ask application code to load the file for `App\Reservation\Court` when the name is first used. Autoloading is a loading mechanism, not dependency injection and not a database lookup. Keep the class’s namespace and file mapping predictable; the details of Composer and PSR-4 belong to Volume VII.

## Minimal Example

The following class models a court without pretending to model the whole reservation system:

```php
declare(strict_types=1);

final class Court
{
    public function __construct(
        private int $id,
        private string $name,
    ) {
        if ($id < 1 || trim($name) === '') {
            throw new InvalidArgumentException('A court needs a positive id and a name.');
        }
    }

    public function id(): int
    {
        return $this->id;
    }

    public function name(): string
    {
        return $this->name;
    }
}
```

The class has one responsibility: representing a valid court. It does not query the database, send an email, or decide whether a time interval conflicts with another reservation. A small class is not incomplete merely because it does not do everything.

## Practical Example

Suppose an application receives a request to reserve a court. A boundary object can make the domain input explicit:

```php
final class ReservationRequest
{
    public function __construct(
        public readonly int $courtId,
        public readonly DateTimeImmutable $startsAt,
        public readonly DateTimeImmutable $endsAt,
    ) {
        if ($courtId < 1) {
            throw new InvalidArgumentException('Court id must be positive.');
        }

        if ($endsAt <= $startsAt) {
            throw new InvalidArgumentException('End must be after start.');
        }
    }
}
```

The class establishes two useful constraints at its boundary: a positive court identifier and a non-empty half-open interval `[startsAt, endsAt)`. Two intervals that meet at an endpoint do not overlap. That choice should be shared with the database query and concurrency strategy; a class cannot prevent a race between two requests by itself.

## Production Example

In production, classes should communicate their role. A domain object, an application service, and a repository may all be classes, but they have different reasons to change:

```php
interface ReservationRepository
{
    public function overlaps(ReservationRequest $request): bool;
    public function save(ReservationRequest $request): void;
}

final class ReservationService
{
    public function __construct(private ReservationRepository $reservations)
    {
    }

    public function reserve(ReservationRequest $request): void
    {
        if ($this->reservations->overlaps($request)) {
            throw new DomainException('The court is already reserved.');
        }

        // The repository/database must still enforce the invariant atomically.
        $this->reservations->save($request);
    }
}
```

The interface shown here is a preview of later chapters. It demonstrates a useful class boundary: the service coordinates a decision, while storage owns database details. The pre-check improves the error message, but it is not a uniqueness guarantee. Two concurrent calls can both observe “free” unless the database transaction or constraint closes that race.

## Bad Example

This class has a name but no coherent boundary:

```php
final class AppManager
{
    public function createUserAndReserveAndSendEmail(array $input): void
    {
        // validate HTTP input, insert users, charge cards,
        // reserve courts, render HTML, and send email
    }
}
```

The problem is not the number of lines. It is the number of unrelated reasons for change and the absence of explicit contracts. Such a class is hard to test without a full environment and hard to retry safely after a partial failure.

## Better Example

Keep the orchestration small and name the boundaries:

```php
final class CreateReservation
{
    public function __construct(
        private ReservationService $reservations,
        private ConfirmationSender $confirmations,
    ) {
    }

    public function handle(ReservationRequest $request): void
    {
        $this->reservations->reserve($request);
        $this->confirmations->send($request);
    }
}
```

This still has a failure window: a reservation may be committed while sending confirmation fails. The solution is not to add more nouns to the class. It is to choose an explicit policy, such as an outbox record in the same transaction and a retryable worker. Class boundaries make that policy easier to locate; they do not supply it automatically.

## Edge Cases

- A class can be declared `final` to prevent inheritance when extension is not part of its contract. The choice is revisited in Chapter 33.
- An object can implement several interfaces, but a class can extend only one parent class. Interfaces and inheritance are later topics.
- A class declaration may contain static members, but static state is shared by the class rather than owned by one object. Treat it as process state and consider its testing and lifecycle implications.
- Anonymous classes are useful for a small one-off implementation and are covered in Chapter 36.
- `class_exists()` and reflection can inspect class availability, but they should not replace ordinary type declarations and clear dependencies.

## Performance

Creating an object allocates and initializes runtime state, so a tight loop over millions of tiny values may have a different memory profile from a packed array of scalars. That does not make arrays universally faster. Measure the actual workload, and first choose the representation that makes the invariant clear.

The larger performance cost is often at the boundary around a class: a query per object, repeated serialization, or an accidental network call in a constructor. Keep I/O visible and batch work where the requirement allows it. An object method that looks like a cheap calculation should not secretly perform a remote request.

## Security

A class is not a trust boundary merely because its properties are private. Validate untrusted input before it becomes domain state, encode output at the output context, and authorize actions at the application boundary. A `User` object created from request data is not proof that the caller may modify that user.

Avoid accepting arbitrary class names from user input and passing them to dynamic instantiation. If a class chooses a file, handler, or strategy, use a fixed allow-list or a typed factory. Keep secrets out of `var_dump()`, exception messages, and object string representations.

## Testing

Test the class’s contract, not its private storage layout:

```php
final class CourtTest
{
    public function invalid_court_is_rejected(): void
    {
        try {
            new Court(0, 'Centre court');
            throw new RuntimeException('Expected InvalidArgumentException.');
        } catch (InvalidArgumentException) {
            // Expected.
        }
    }

    public function valid_court_exposes_its_identity(): void
    {
        $court = new Court(3, 'Centre court');

        assert($court->id() === 3);
        assert($court->name() === 'Centre court');
    }
}
```

In PHPUnit, use an expected exception assertion or `expectException()`. Also test the integration boundary separately: a unit test should not need a real database to prove that an invalid court is rejected, while a repository integration test should prove the SQL and transaction behavior.

## Common Mistakes

- Treating a class as a synonym for a database table.
- Creating an `*Manager` or `*Helper` class with unrelated operations.
- Using public mutable properties for every field and calling the result encapsulation.
- Hiding I/O in constructors or getters.
- Assuming a type declaration proves a domain invariant.
- Assuming an object survives the end of a web request.
- Introducing an interface only to make a class “look testable,” without a real replaceable boundary.

## Senior Engineer Thinking

Before creating a class, write down the invariant or decision it owns. Then ask which caller needs that guarantee, what data it must have, and which side effects belong outside it. If the answer is “this is just a record passed between layers,” a small immutable data object may be enough. If the answer is “this object protects a rule through operations,” a richer domain object may be justified.

The best class design is not the one with the most methods. It is the one whose public surface makes invalid or ambiguous use difficult, while leaving infrastructure concerns at visible boundaries.

## Exercises

1. Create a `Player` class with a positive integer id and a non-empty display name. Decide whether the name should be normalized, and test both accepted and rejected inputs.
2. Split a procedural `reserveAndEmail()` function into an application class, a reservation boundary, and a notification boundary. List the failure states after each external side effect.
3. Implement an `Interval` class for half-open time intervals. State its overlap rule in prose, code, and two tests.
4. Measure memory and runtime for one million scalar rows versus one million small objects in the PHP version used by your application. Record what the benchmark does and does not prove.

## Review Questions

1. What is the difference between a class definition and an object instance?
2. Which guarantees does a class type declaration provide, and which domain guarantees does it not provide?
3. Why should a database uniqueness or concurrency rule not be delegated to a PHP class alone?
4. When does a class have too many responsibilities?
5. Why is hidden I/O in a method or constructor operationally dangerous?

## Summary

A class is a named type and boundary for state, behavior, and invariants. Objects are instances of that definition. Use classes to make contracts and responsibilities explicit, not to turn every array or function into ceremony. Keep validation and domain decisions distinct from persistence, transport, and notification side effects. The next chapter examines the runtime identity and assignment behavior of the objects created from these classes.
