---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 10
title: Types
slug: types
status: complete
summary: ../../_ai/chapter-summaries/010-types-summary.md
---

# Chapter 10 — Types

## Why This Matters

Every PHP expression has a runtime type. A value may be `int`, `string`, `array`, an object, or one of PHP’s other supported types even when the source code did not declare a type. The type tells the runtime what operations and representations are available. It does not, by itself, tell the application what the value means.

The difference is important:

```text
string        → a runtime category
email address → a domain meaning
```

Both an email address and an SQL fragment can be strings. A type declaration can require a string; it cannot make the string a valid email or safe SQL. Reliable PHP code uses native types to make contracts visible, then validates domain rules at the boundary where those rules belong.

Modern PHP gives us a useful middle ground between completely untyped code and a language with no runtime flexibility. We can type function parameters, return values, properties, class constants, and declarations with unions, intersections, nullable types, enums, and object types. We can also use static analysis annotations for shapes and generic-like collections that PHP’s runtime type system does not express directly.

## Mental Model

Separate three questions:

1. What type does this value have at runtime?
2. What types does this function or property accept?
3. What semantic and security rules must the value satisfy?

For example:

```php
function reserve(int $courtId, DateTimeImmutable $startsAt): void
{
    // Native types establish part of the call contract.
}
```

This rejects a non-integer court identifier and a non-date object at the function boundary, subject to PHP’s scalar coercion rules and the caller’s `strict_types` setting. It does not establish that the court exists, the caller may reserve it, the date is in the permitted future, or the interval ends after it starts.

Think of a type declaration as a gate, not a complete validator:

```text
external representation
  → parse and validate shape
  → construct typed value
  → enforce domain invariant
  → perform application work
```

The next two chapters examine type juggling and strict typing. This chapter establishes the vocabulary and the available type forms.

## Core Concept

### The built-in value types

Modern PHP values can have these broad categories:

| Type | What it represents | Typical use |
| --- | --- | --- |
| `null` | Absence of a value | Optional result or explicit empty state |
| `bool` | `true` or `false` | A genuine two-state decision |
| `int` | A signed integer within the platform’s integer range | Counts, identifiers, minor currency units |
| `float` | A binary floating-point number | Measurements or approximations, not exact money |
| `string` | A sequence of bytes, commonly UTF-8 text by convention | Text, encoded data, identifiers |
| `array` | An ordered map of keys to values | Lists, maps, records, mixed legacy structures |
| `object` | An instance with identity and behavior | Domain objects, services, dates, exceptions |
| `resource` | A handle to an external resource managed by an extension | Open streams or extension-specific handles |
| `callable` | A value PHP can invoke | Function names, closures, callable objects |

`get_debug_type()` gives a useful human-facing type name, while `var_dump()` shows type and representative value:

```php
$values = [null, false, 42, 3.14, '42', [], new stdClass()];

foreach ($values as $value) {
    echo get_debug_type($value), PHP_EOL;
}
```

Use `is_int()`, `is_string()`, `is_array()`, `is_object()`, and the other `is_*` functions when the program needs to branch on a runtime type. `gettype()` exists for compatibility, but `get_debug_type()` generally produces more useful names for modern code.

### Scalar values are not interchangeable

An integer, float, string, and boolean can participate in expressions in ways that involve conversion. That behavior is type juggling and is covered in Chapter 11. Do not design a domain around the assumption that values which can be converted are equivalent.

Money is a classic example. A float is an approximation in binary floating point:

```php
$total = 0.1 + 0.2;
var_dump($total); // Not a reliable representation of exact 0.30.
```

For currency, store minor units as integers when the currency rules permit it, or use a decimal representation/library designed for the required precision. A type of `float` describes representation; it does not promise exact arithmetic.

### `array` is one runtime type with several designs

PHP arrays can represent a list:

```php
$surfaces = ['clay', 'hard', 'grass'];
```

a map:

```php
$courtById = [3 => 'Court 3', 8 => 'Court 8'];
```

or a record:

```php
$court = ['id' => 3, 'surface' => 'clay'];
```

All three values have runtime type `array`, but their invariants differ. Native PHP does not declare a runtime element type such as `list<string>` or `array<int, Court>` in a function signature. PHPDoc or analyzer syntax can communicate those shapes to tools:

```php
/** @param list<string> $surfaces */
function printSurfaces(array $surfaces): void
{
    foreach ($surfaces as $surface) {
        echo $surface, PHP_EOL;
    }
}
```

When a record has important invariants, an object or value object can make ownership and validation clearer than an unstructured array. The right choice depends on the data’s lifetime, transformation needs, and boundary.

### Objects have class identity

An object can be checked against its class or an interface it implements:

```php
interface Clock
{
    public function now(): DateTimeImmutable;
}

final class SystemClock implements Clock
{
    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable();
    }
}

function currentInstant(Clock $clock): DateTimeImmutable
{
    return $clock->now();
}
```

The parameter accepts any object that satisfies the `Clock` contract, not one specific implementation. This is useful at a dependency boundary because a test clock can implement the same interface. Do not add an interface merely to provide a second name for a class when no substitution or boundary exists.

### `null` is a type and a design decision

An optional value can be declared nullable:

```php
function findCourtName(int $id): ?string
{
    return $id === 3 ? 'Centre Court' : null;
}
```

`?string` is shorthand for `string|null`. The caller must decide what absence means: not found, not loaded, not authorized to disclose, or genuinely empty. Returning `null` without documenting that distinction moves ambiguity to every caller.

When a result has several meaningful outcomes, a result object or enum can be clearer than a union containing `null` and a value. The simplest representation is preferable only when it preserves the domain distinction.

## How It Works

PHP is dynamically typed: values carry runtime types, and variables can be rebound to values of different types. Type declarations add runtime checks at selected boundaries rather than turning every variable into a compile-time-only slot.

Consider a parameter declaration:

```php
function courtNumber(int $value): int
{
    return $value;
}
```

When the function is called, PHP checks the argument against the declaration. If it cannot accept the value under the current typing mode, it throws `TypeError`. The return value is checked against the return declaration as the function returns. A typed property is checked when a value is assigned to it, and a typed class constant is checked by the declaration.

Declarations therefore move failures closer to the boundary where an incorrect value enters. They do not eliminate runtime checks or make invalid input safe. A type error is a programming or contract failure; ordinary user input usually needs a validation error that the application can explain without exposing internals.

## What PHP Does

PHP supports these declaration forms in modern versions:

### Scalar and built-in declarations

```php
function label(string $surface, int $number, bool $isOpen): string
{
    return $isOpen ? "{$surface} court {$number}" : 'Closed';
}
```

The special `void` return type means the function does not return a value. A function declared `never` does not return at all: it throws, exits, or otherwise never reaches its caller. `never` is return-only and cannot be part of a union.

`mixed` is the broad type that can represent any PHP value, including `null`. It is useful at a deliberately untyped boundary such as a generic serializer hook, but it gives callers little protection. Narrow a `mixed` value as soon as the program knows what it expects.

`iterable` is an alias for `array|Traversable`. It describes values that can be iterated, not necessarily values that support random access or count operations. `callable` describes invocability, but it cannot be used as a property type; a `Closure` property is often appropriate when a stored closure is required.

### Class, interface, enum, and special object types

Class and interface types communicate behavior and identity. Enums provide a finite set of named cases:

```php
enum Surface: string
{
    case Clay = 'clay';
    case Hard = 'hard';
    case Grass = 'grass';
}

function courtSurfaceLabel(Surface $surface): string
{
    return ucfirst($surface->value);
}

$surface = Surface::Clay;
```

The enum case is an object of enum type, while its backed value is a string. `Surface::Clay` and `'clay'` are not interchangeable at a typed boundary. Converting untrusted input to the enum is a validation step:

```php
$surface = Surface::tryFrom($rawSurface);

if ($surface === null) {
    throw new InvalidArgumentException('Unsupported surface.');
}
```

`self`, `parent`, and `static` are relative class types used in inheritance-aware declarations. `static` as a return type expresses late-static return behavior; it is not the same as the `static` property modifier.

### Union types

A union accepts one of several alternatives:

```php
function formatIdentifier(int|string $identifier): string
{
    return (string) $identifier;
}
```

Use a union when the alternatives are genuinely part of the API contract and the function can handle each alternative coherently. A union that exists only because callers pass inconsistent data is often a signal to normalize at the boundary instead.

Nullable syntax is a common union:

```php
function findReservation(int $id): Reservation|null
{
    // ...
}
```

The `?Reservation` spelling means the same thing for a single named type. Modern PHP also supports standalone `null`, `false`, and `true` types in appropriate declarations, as well as literal `false` in unions useful for some legacy APIs. Prefer a result model when a false-like sentinel would make success and failure ambiguous.

### Intersection and DNF types

An intersection requires one object to satisfy every listed class or interface contract:

```php
interface Renderable
{
    public function render(): string;
}

interface Cacheable
{
    public function cacheKey(): string;
}

function cacheRendered(Renderable&Cacheable $value): string
{
    return $value->cacheKey() . ':' . $value->render();
}
```

Intersection members are class types—usually interfaces—and the value must satisfy all of them. A union is “this or that”; an intersection is “this and that.”

PHP also supports disjunctive normal form (DNF), combining unions and intersections with parentheses:

```php
function consume(Renderable|(Cacheable&Stringable) $value): void
{
    // Accept Renderable, or an object that is both Cacheable and Stringable.
}
```

DNF is expressive but can make an API hard to read. Introduce a named interface or value object when the combination represents an important domain concept.

### Typed properties and constants

Typed properties make object state explicit:

```php
final class Reservation
{
    public int $courtId;
    public DateTimeImmutable $startsAt;

    public function __construct(int $courtId, DateTimeImmutable $startsAt)
    {
        $this->courtId = $courtId;
        $this->startsAt = $startsAt;
    }
}
```

A typed property without a default is uninitialized until construction or another assignment initializes it. Reading it before initialization throws an `Error`; it is not automatically `null`. If `null` is valid, declare it nullable and initialize it intentionally. Typed class constants are available in current PHP 8.x and are checked as part of the declaration.

## What Zend Does

Zend stores a runtime type alongside each value, checks declarations at the relevant call, return, property, or constant boundary, and raises engine-level errors such as `TypeError` or `Error` when the contract is violated. The internal zval representation and type checks are developed in Volume IV; the source-level lesson is that a declaration changes where invalid values are rejected, not whether the application needs domain validation.

Scalar declarations may involve coercion unless strict typing applies at the call site. An `int` can be accepted where a `float` is declared even in strict mode; other scalar conversions differ between coercive and strict calls. Chapter 11 explains the conversion rules and Chapter 12 explains `declare(strict_types=1)` in detail. Do not infer strict behavior from a type declaration alone.

## Minimal Example

Use a small typed value object to represent a valid positive court identifier:

```php
<?php

declare(strict_types=1);

final readonly class CourtId
{
    public function __construct(public int $value)
    {
        if ($value < 1) {
            throw new InvalidArgumentException('Court ID must be positive.');
        }
    }
}

$courtId = new CourtId(3);
echo $courtId->value, PHP_EOL;
```

The native `int` declaration prevents a non-integer object state, while the constructor enforces the domain rule that an identifier must be positive. Neither rule says that court 3 exists; that belongs to a repository or database boundary.

## Practical Example

Parse an external reservation payload into typed values before applying the business rule:

```php
<?php

declare(strict_types=1);

final readonly class ReservationInput
{
    public function __construct(
        public CourtId $courtId,
        public DateTimeImmutable $startsAt,
        public DateTimeImmutable $endsAt,
    ) {
        if ($this->startsAt >= $this->endsAt) {
            throw new InvalidArgumentException('Reservation must have positive duration.');
        }
    }

    /** @param array{court_id: int|string, starts_at: string, ends_at: string} $payload */
    public static function fromPayload(array $payload): self
    {
        $courtId = $payload['court_id'] ?? null;

        if (is_string($courtId) && ctype_digit($courtId)) {
            $courtId = (int) $courtId;
        }

        if (!is_int($courtId)) {
            throw new InvalidArgumentException('court_id must be an integer.');
        }

        try {
            $startsAt = new DateTimeImmutable($payload['starts_at']);
            $endsAt = new DateTimeImmutable($payload['ends_at']);
        } catch (Exception $exception) {
            throw new InvalidArgumentException('Reservation dates are invalid.', 0, $exception);
        }

        return new self(new CourtId($courtId), $startsAt, $endsAt);
    }
}
```

The PHPDoc array shape helps static analysis, while runtime checks still handle untrusted data. `DateTimeImmutable` parsing needs a documented format and time-zone policy in a real API; accepting every format the constructor happens to recognize may be too permissive. Types make the next layer simpler because it no longer handles arbitrary arrays and date strings.

## Production Example

A typed dependency boundary can be shared by an HTTP handler, a CLI command, and a test:

```php
interface ReservationRepository
{
    public function find(int $id): ?Reservation;
}

final class ReservationService
{
    public function __construct(private ReservationRepository $repository)
    {
    }

    public function labelFor(int $id): string
    {
        $reservation = $this->repository->find($id);

        if ($reservation === null) {
            throw new RuntimeException('Reservation was not found.');
        }

        return sprintf('Court %d', $reservation->courtId);
    }
}
```

The nullable return type forces the not-found decision to be visible. The repository interface is justified because it marks a persistence boundary and permits a real integration implementation or a focused test double. The `int` type does not authorize the caller to see reservation `id`; authorization still belongs to the application policy.

## Bad Example

This function has a type declaration but still has a weak contract:

```php
function createPayment(string $amount, string $currency): void
{
    $sql = "INSERT INTO payments (amount, currency) VALUES ('$amount', '$currency')";
    // ...
}
```

The strings may be malformed, the amount may contain an unexpected precision, the currency may not be supported, and the SQL is injectable. A declaration of `string` confirms only a runtime category. It does not validate money, currency, authorization, or SQL safety.

Another failure is overusing `mixed`:

```php
function transform(mixed $input): mixed
{
    return $input;
}
```

This may be an honest pass-through at a generic boundary, but it gives the rest of the application no useful contract. If the function is supposed to transform a reservation payload, its input and output should say so.

## Better Example

Represent money in minor units and validate the currency at construction:

```php
enum Currency: string
{
    case EUR = 'EUR';
    case USD = 'USD';
}

final readonly class Money
{
    public function __construct(
        public int $minorUnits,
        public Currency $currency,
    ) {
        if ($minorUnits < 0) {
            throw new InvalidArgumentException('Money cannot be negative here.');
        }
    }
}

function createPayment(Money $amount): void
{
    // Bind $amount->minorUnits and $amount->currency->value in a prepared query.
}
```

Now the function cannot accidentally receive an arbitrary string in place of the payment amount. The value object still does not prove that the account has funds or that the current user may charge it; those are separate application and external-system rules.

## Edge Cases

- `?T` means `T|null`; it is not the same as an optional parameter. A required nullable parameter must still be passed.
- A typed property without a default is uninitialized, not null. Reading it before initialization throws an `Error`.
- `mixed` includes every PHP value, including `null`. Narrow it at the boundary and document why it is necessary.
- `void` means no value is returned; `never` means control never returns. Returning from a `never` function violates its contract.
- `iterable` includes arrays and `Traversable` objects, but it does not promise indexing or `count()`.
- `callable` is allowed for parameters and returns but not as a property type. Use `Closure` when a closure must be stored as a property.
- `resource` values exist at runtime but cannot be used as userland type declarations. Wrap resources behind an object or keep them at an extension boundary.
- `int` has a platform-dependent range. Do not assume an external identifier or file size always fits; validate before conversion.
- `float` is not an exact decimal type. Use integer minor units or a decimal strategy for money.
- Native `array` declarations do not constrain keys or element types. Use objects, PHPDoc shapes, and static analysis where those invariants matter.
- Union declarations cannot contain redundant alternatives such as `bool|false`; intersection members must be class or interface types.
- DNF types require parentheses around intersection alternatives and are more difficult to communicate than a named abstraction.
- Enums and backed scalar values are different types. Convert at input boundaries with `from()` or `tryFrom()` and handle invalid cases.

## Performance

Type declarations usually improve performance reasoning more than they serve as a micro-optimization. They reduce the number of states a function must handle and can let static tools find mistakes before runtime. They do not make a database query, network call, or object allocation free.

The representation still matters. A large PHP array can consume far more memory than a compact serialized or database representation; an object graph can retain more state than a scalar identifier; converting a million strings to objects may cost CPU and memory. Choose types and structures according to data volume and lifetime, then measure.

Avoid converting values repeatedly at every layer. Normalize a request once at the boundary, pass the typed value through the application, and serialize only when crossing a transport or persistence boundary. That reduces duplicated parsing and makes the cost visible.

## Security

Types are a defense-in-depth tool, not an authorization system:

- `int $userId` does not prove that the current actor owns that user;
- `string $html` does not mean HTML is safe to render;
- `array $filters` does not make filter keys safe SQL identifiers;
- `UploadedFile` does not prove that content is safe to store or execute;
- `DateTimeImmutable` does not prove the time zone or business range is acceptable.

Parse and validate untrusted representations before constructing domain values. Use allow-lists for enum values and dynamic operations. Parameterize SQL values, escape output for its context, and avoid exposing raw `TypeError` messages or class names to clients. A type error may contain argument names and implementation details that belong in logs, not public responses.

## Database Interaction

Database rows are external representations. A column declared `INTEGER` does not automatically arrive in PHP with the same type for every driver, fetch mode, or query. Decide where conversion happens and test the chosen PDO/driver configuration.

When writing typed values, bind the representation required by the database and keep the invariant in both layers where appropriate:

```php
$statement = $pdo->prepare(
    'INSERT INTO payments (amount_minor, currency) VALUES (:amount, :currency)',
);
$statement->execute([
    'amount' => $money->minorUnits,
    'currency' => $money->currency->value,
]);
```

The PHP value object prevents invalid negative amounts in this process, while a database constraint can protect the invariant against other writers. Types and constraints are complementary. A typed property cannot stop a direct SQL client from inserting invalid data.

## Concurrency

Type declarations are checked within one call and process; they do not synchronize multiple workers. Two requests can each receive a valid `int $courtId` and still race to reserve the same court. Types describe the shape of the command, while a transaction, constraint, lock, or atomic update protects the shared state transition.

Likewise, two workers can deserialize the same message into equally valid typed objects. Idempotency and duplicate-delivery handling remain application responsibilities. A type-correct command can still be stale, unauthorized, or repeated.

## Testing

Test both type contracts and semantic contracts:

- call typed functions with valid and invalid values and assert the intended `TypeError` behavior;
- test typed-property initialization and invalid assignment where those are part of the API;
- test nullable results for found and not-found cases;
- test enum conversion with every supported case and an invalid value;
- test value-object invariants such as positive identifiers and nonnegative money;
- test array shapes at the adapter boundary, including missing keys and wrong runtime types;
- test database serialization and hydration with the actual driver configuration;
- test authorization separately from type acceptance.

Use static analysis to check PHPDoc shapes, generic-like collections, unreachable branches, and possible null values. A passing type check is not a substitute for integration tests of time zones, database constraints, serialization, or concurrent transitions.

## Common Mistakes

- Treating a native type as a complete domain validator.
- Using `float` for exact monetary arithmetic.
- Returning `null` without defining what absence means.
- Using `mixed` to postpone every design decision.
- Assuming `array` means list, map, or record without documenting the shape.
- Reading an uninitialized typed property and expecting `null`.
- Using an interface for every class without a real substitution boundary.
- Assuming strict scalar behavior without understanding the caller’s `strict_types` setting.
- Trusting database driver output types without testing the fetch configuration.
- Believing a typed command prevents authorization failures, duplicate delivery, or race conditions.

## Senior Engineer Thinking

When choosing a type, ask:

1. What invalid states does this declaration prevent?
2. Which semantic states remain possible even after the type check?
3. Should this value be normalized once at a boundary or repeatedly converted?
4. Does `null` represent a real domain state, or is it hiding a missing result?
5. Would an object, enum, or named interface communicate more than `array` or `string`?
6. Is a union describing a stable API or compensating for inconsistent callers?
7. Which invariant belongs in PHP, static analysis, the database, or an external system?

Good typing narrows the program’s state space without pretending that types solve every problem. The strongest contracts combine native declarations, semantic validation, explicit ownership, database constraints, and tests at the boundaries where failure matters.

## Exercises

1. Implement a `Money` value object using integer minor units and a backed `Currency` enum. Test invalid negative values and unsupported currencies.
2. Write a function using `int|string` and then refactor the callers so it can accept only one normalized type. Explain which design is clearer and why.
3. Create a class with an uninitialized typed property. Observe the failure when reading it, then decide whether the property should be initialized, nullable, or removed.
4. Define two interfaces and write a function accepting an intersection type. Implement one class that satisfies both and one that satisfies only one; test the boundary.
5. Take an API payload represented as `array<string, mixed>` and convert it into a typed value object. List the rules that native type declarations cannot express.

## Review Questions

1. What is the difference between a runtime type and a domain invariant?
2. Why is `string` not enough to represent a safe email, SQL fragment, or HTML value?
3. What does `?T` mean?
4. What are the differences between `mixed`, `void`, and `never`?
5. Why does `array` not communicate whether a value is a list, map, or record?
6. When is an interface type more useful than a concrete class type?
7. What does a union express, and what does an intersection express?
8. Why do PHP types not replace database constraints or authorization checks?
9. What happens when a typed property is read before initialization?
10. Why should money generally not be represented by `float`?

## Summary

PHP is dynamically typed, but modern PHP supports strong runtime contracts at function, return, property, constant, class, interface, and enum boundaries. Built-in types describe runtime categories; objects, interfaces, enums, unions, intersections, nullable types, `mixed`, `void`, and `never` let APIs express more precise contracts. Native types do not validate domain meaning, authorization, output safety, database invariants, or concurrency. Normalize external data once, use value objects and named abstractions when they clarify invariants, combine declarations with static analysis and database constraints, and treat type errors as contract failures rather than user-facing validation.

