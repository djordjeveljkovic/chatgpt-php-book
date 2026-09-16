---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 26
title: Constructors
slug: constructors
status: complete
summary: ../../_ai/chapter-summaries/026-constructors-summary.md
---

# Chapter 26 — Constructors

## Why This Matters

A constructor is the first opportunity to prevent an invalid object from existing. If a `TimeSlot` can be created with an end before its start, every method must defend against that state forever. If a service can be constructed without its repository, the failure appears later as a null call or hidden global lookup.

Constructors are therefore boundaries for required state and required dependencies. They are not a good place for every piece of application work. A constructor that opens connections, performs queries, sends messages, or reads request globals makes object creation surprising, slow, and difficult to retry.

## Mental Model

`new ClassName(arguments)` allocates an object and invokes its `__construct()` method when the class defines one:

```php
final class Point
{
    public function __construct(
        public readonly int $x,
        public readonly int $y,
    ) {
    }
}

$point = new Point(3, 4);
```

The constructor’s job is to establish the state needed for the object’s public methods to be safe. That usually means assigning required properties, validating local invariants, normalizing representation, and storing dependencies. It should leave orchestration to an application method or service.

## Core Concept

An object should be usable immediately after construction:

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

    public function durationInMinutes(): int
    {
        return intdiv($this->endsAt->getTimestamp() - $this->startsAt->getTimestamp(), 60);
    }
}
```

After `new TimeSlot(...)` succeeds, `durationInMinutes()` has a meaningful precondition. The constructor throws before returning an unusable object. This is different from accepting invalid input and relying on every caller to remember a later `validate()` call.

Validation should fit the object’s responsibility. A `TimeSlot` can validate ordering. It should not check whether a court is available in a database; availability is a concurrent application/database decision.

## How It Works

The constructor is an ordinary method with the special name `__construct`. It can accept required and optional arguments, type declarations, defaults, and named arguments. If a child class defines its own constructor, the parent constructor is not called implicitly; the child must call `parent::__construct()` when parent initialization is required. A child without a constructor can inherit the parent constructor.

Old-style constructors named after the class are obsolete and should not be used in new code. Always use `__construct()`.

Constructor property promotion, introduced in PHP 8.0, combines a constructor parameter and a property declaration:

```php
final class Point
{
    public function __construct(
        private int $x,
        private int $y,
    ) {
    }
}
```

Promotion is syntax for a common assignment pattern; it is not a different object model. Additional statements in the constructor body run after promoted arguments have been assigned. You can mix promoted and ordinary parameters. Do not promote a dependency or property merely to shorten code if its validation or lifecycle deserves a named operation.

## What PHP Does

PHP evaluates constructor arguments before entering the constructor. Type errors can therefore occur before the body runs. If the constructor throws, the caller does not receive a successfully constructed object; any external side effect performed before the throw may still have happened, which is one reason to keep constructors side-effect-light.

A constructor can be private or protected. This prevents arbitrary external `new` calls and lets named static factories control creation:

```php
final class EmailAddress
{
    private function __construct(private string $value)
    {
    }

    public static function fromString(string $value): self
    {
        $value = strtolower(trim($value));

        if (filter_var($value, FILTER_VALIDATE_EMAIL) === false) {
            throw new InvalidArgumentException('Invalid email address.');
        }

        return new self($value);
    }
}
```

Factories are useful for multiple input representations, parsing, normalization, or named alternatives such as `fromJson()` and `forGuest()`. A factory should not become a hiding place for network calls that make “creation” unpredictable.

## Minimal Example

Required dependencies belong in the constructor:

```php
interface ReservationRepository
{
    public function save(ReservationRequest $request): void;
}

final class ReservationWriter
{
    public function __construct(private ReservationRepository $repository)
    {
    }

    public function write(ReservationRequest $request): void
    {
        $this->repository->save($request);
    }
}
```

The object cannot be created without a repository. This is dependency injection: the caller supplies the collaborator, making the dependency visible and replaceable in a unit test. It does not mean every dependency needs an interface; use a concrete class when substitution is not a real requirement.

## Practical Example

Normalize and validate a value object in its constructor:

```php
final class Money
{
    public readonly int $minorUnits;
    public readonly string $currency;

    public function __construct(int $minorUnits, string $currency)
    {
        $currency = strtoupper(trim($currency));

        if ($minorUnits < 0 || !preg_match('/^[A-Z]{3}$/', $currency)) {
            throw new InvalidArgumentException('Invalid money value.');
        }

        $this->minorUnits = $minorUnits;
        $this->currency = $currency;
    }
}
```

The constructor accepts a broader input representation for currency casing but stores one normalized representation. It rejects negative amounts because this value object models non-negative prices. If refunds or account balances need negative values, that is a different constraint and perhaps a different type. Do not copy validation rules without checking the requirement.

## Production Example

A service constructor should assemble dependencies, while work happens in a method:

```php
interface AvailabilityChecker
{
    public function reserveIfFree(ReservationRequest $request): string;
}

interface ConfirmationQueue
{
    public function enqueue(string $reservationId): void;
}

final class ReserveCourt
{
    public function __construct(
        private AvailabilityChecker $availability,
        private ConfirmationQueue $confirmations,
    ) {
    }

    public function handle(ReservationRequest $request): string
    {
        $id = $this->availability->reserveIfFree($request);
        $this->confirmations->enqueue($id);

        return $id;
    }
}
```

The constructor does no I/O. `handle()` makes the side-effect order visible. In production, `reserveIfFree()` should use a transaction or database constraint appropriate to the interval invariant, and enqueueing should be made durable with an outbox or equivalent if losing the confirmation message is unacceptable. Constructor design supports that architecture by keeping dependencies explicit; it cannot make two external operations atomic by itself.

## Bad Example

```php
final class ReservationService
{
    private PDO $pdo;

    public function __construct()
    {
        $this->pdo = new PDO($_ENV['DATABASE_DSN']);
        $this->pdo->exec('SET time_zone = \'UTC\'');
        $this->warmCache();
    }
}
```

Every test and every caller now creates a database connection and performs I/O merely to obtain the object. Construction can fail because of the network, configuration, or a cache dependency before the caller has chosen an error policy. It also makes a harmless dependency graph expensive during application startup.

## Better Example

Inject a configured collaborator and make initialization explicit at the composition root:

```php
final class ReservationService
{
    public function __construct(private PDO $connection)
    {
    }

    public function reserve(ReservationRequest $request): void
    {
        // The method owns the transaction boundary or delegates to a repository.
    }
}

$pdo = createConfiguredPdo($_ENV['DATABASE_DSN']);
$service = new ReservationService($pdo);
```

The composition root decides how to create and configure the connection. The service declares what it needs but does not decide how the entire process obtains configuration. In tests, a test connection or repository can be supplied according to the test’s scope.

## Constructor Design Choices

Use a required constructor parameter when the object cannot work without the value. Use a default only when there is a genuine safe default, not to make an invalid call compile. Prefer a value object when several primitive parameters must agree. Prefer a named factory when creation has a distinct language or input format.

Do not use a constructor to model an asynchronous workflow. Construct a command or service, then call a method that performs the workflow. Do not call overridable methods from a parent constructor: a child may observe properties that have not yet been initialized, and the order becomes part of a fragile inheritance contract.

## Edge Cases

- A class with no constructor can be instantiated with no arguments; an empty constructor is usually unnecessary unless it documents or controls visibility.
- A promoted property’s default belongs to the constructor parameter’s call contract; the property is initialized when construction occurs.
- `new` in default initializers is available in modern PHP 8.1+ constant-expression contexts, but explicit construction is often clearer for mutable or request-dependent dependencies.
- Constructors cannot be overloaded by signature. Use named factories for alternate creation paths.
- A private constructor does not prevent the class from creating instances internally; it changes the public creation API.
- Destructors are not a reliable inverse of constructor work for transactions or remote effects. Chapter 27 covers cleanup and shutdown behavior.

## Performance

Keep constructors cheap enough that dependency graphs can be assembled predictably. Expensive work should be explicit, measurable, and placed at the operation that needs it. Lazy initialization can help for genuinely optional expensive resources, but it introduces state and concurrency questions; do not add it merely to conceal a poor boundary.

For large imports, constructing one object per row may be appropriate for domain validation but expensive if the object graph is retained. Stream batches, release references, and measure worker memory. Constructor promotion reduces source code, not the runtime cost of allocating and initializing the object.

## Security

Do not pass raw user-controlled configuration into constructors that choose classes, files, commands, or network destinations without allow-list validation. Validate secrets and credentials at the correct configuration boundary, but avoid logging constructor arguments automatically. A constructor can establish “this token is syntactically valid”; authorization and rotation remain operational concerns.

Never make a constructor trust that an id implies access. A `Reservation` constructed from a client-provided id still needs an authorization check before mutation or disclosure.

## Testing

Constructor tests should prove both successful invariants and rejected inputs:

```php
function test_money_normalizes_currency(): void
{
    $money = new Money(1250, ' eur ');

    assert($money->minorUnits === 1250);
    assert($money->currency === 'EUR');
}

function test_money_rejects_negative_prices(): void
{
    try {
        new Money(-1, 'EUR');
        throw new RuntimeException('Expected InvalidArgumentException.');
    } catch (InvalidArgumentException) {
        // Expected.
    }
}

function test_service_receives_its_dependency(): void
{
    $repository = new class implements ReservationRepository {
        public bool $saved = false;
        public function save(ReservationRequest $request): void { $this->saved = true; }
    };

    $writer = new ReservationWriter($repository);
    // Pass a valid ReservationRequest and assert that write delegates once.
}
```

Do not write tests that assert an implementation used constructor promotion instead of explicit properties. Test the resulting public contract. For a service with real I/O, unit-test orchestration with a focused fake and integration-test configuration, transactions, and database errors separately.

## Common Mistakes

- Leaving required typed properties uninitialized after construction.
- Using nullable properties to avoid deciding whether a dependency is required.
- Performing database, network, cache, or message work in every constructor.
- Calling overridable methods from a constructor.
- Adding defaults that create invalid or ambiguous objects.
- Treating constructor promotion as a substitute for validation.
- Assuming constructor success makes a resource operation durable or authorized.

## Senior Engineer Thinking

Ask what must be true immediately after `new` returns. Put those facts in the constructor or a named factory, and make every later state transition explicit. Then ask whether construction is deterministic: can it be performed in a unit test without the network, and can the application choose an appropriate failure/retry policy for expensive work?

Constructor signatures also reveal coupling. A service requiring twelve collaborators may be doing too much; a value object requiring many primitives may need a more coherent data type. Do not fix those smells mechanically—trace the responsibilities and boundaries first.

## Exercises

1. Build a `TimeSlot` constructor that rejects zero-length and reversed intervals. Add tests for timezone-aware timestamps.
2. Refactor a service that constructs its own PDO, clock, and mailer so those dependencies are injected. Identify the composition root.
3. Create `EmailAddress::fromString()` and `EmailAddress::fromParts()` factories with one private constructor. Decide where normalization belongs.
4. Write a child class with a parent constructor and demonstrate when `parent::__construct()` is required. Then replace the hierarchy with composition and compare the initialization contracts.

## Review Questions

1. What should be true when a constructor successfully returns?
2. Why should a constructor usually avoid external side effects?
3. What did constructor property promotion, introduced in PHP 8.0, change syntactically?
4. When is a named factory clearer than one public constructor?
5. Why does injecting a dependency improve testability and failure policy?

## Summary

Constructors establish required state, validate local invariants, normalize values, and store explicit dependencies. Keep them deterministic and light; perform workflows and external I/O in named methods. Use promotion to reduce assignment boilerplate, not to skip design. Use factories for meaningful alternate creation paths, and remember that parent constructors, database constraints, authorization, and durable side effects still require explicit engineering.
