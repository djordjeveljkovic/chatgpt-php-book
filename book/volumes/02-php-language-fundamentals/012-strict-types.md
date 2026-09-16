---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 12
title: Strict Types
slug: strict-types
status: complete
summary: ../../_ai/chapter-summaries/012-strict-types-summary.md
---

# Chapter 12 — Strict Types

## Why This Matters

Type declarations describe a function's contract, but a contract is only useful if the boundary enforces the intended meaning. PHP supports scalar type declarations with coercive behavior by default and offers a per-file strict mode for calls made from a file. Understanding that boundary prevents two opposite mistakes: believing PHP is statically typed, or believing type declarations are merely documentation.

Strict types are not a switch that transforms an entire application into another language. They are a precise tool for making scalar function calls and returns reject values that should not be silently converted. Used consistently, they make bad input fail near its source. Used carelessly, they can expose hidden coupling during migration or create a false sense that untyped input is safe.

## Mental Model

There are three layers of type discipline:

```text
Input representation → normalization/validation → typed function boundary
```

`declare(strict_types=1);` controls scalar coercion for calls made from that source file. It does not make ordinary variables statically typed, does not validate array contents, and does not automatically propagate to every included or called file.

The call site matters. A function can be declared in one file and called from a strict file or a coercive file. The caller's file determines how scalar arguments are handled. This is one of PHP's most important “the file where the action occurs” rules.

## Core Concept

### Type declarations are executable contracts

```php
function reserve(int $courtId, DateTimeImmutable $startsAt): void
{
    // The body begins only after the parameter contract is checked.
}
```

The object parameter must be a `DateTimeImmutable` (or compatible subtype), and the integer parameter must satisfy PHP's scalar boundary rules. If a value cannot be accepted, PHP throws `TypeError` before the function body runs.

Return types protect callers in the other direction:

```php
function remainingSlots(int $used, int $capacity): int
{
    return $capacity - $used;
}
```

A return declaration protects the boundary from an accidental value of the wrong type. It does not prove that the result is sensible: `-4` is still an integer. Domain validation remains the function's responsibility.

### Coercive calls and strict calls

With no strict declaration in the caller, PHP may coerce a scalar argument when a compatible conversion exists:

```php
function addOne(int $value): int
{
    return $value + 1;
}

addOne('41'); // accepted by a coercive caller; the function receives 41
```

From a strict caller:

```php
declare(strict_types=1);

addOne('41'); // TypeError: string is not accepted as int
```

Strict mode does not reject every related numeric value in every direction. PHP's declared-type rules include specific exceptions, such as allowing an integer where a float is expected. Read the contract as PHP defines it; do not infer a complete mathematical subtype system from the syntax.

Strict typing is defined for scalar declarations. Class, interface, enum, and array types are checked according to their own rules rather than being converted by scalar strictness.

### The declaration is per file

The directive must appear at the top of the file, before ordinary executable code:

```php
<?php

declare(strict_types=1);
```

It applies to calls made from that file, including calls to functions declared elsewhere. If a non-strict file calls a function declared in a strict file, the non-strict caller's coercive preference is used for the argument call. Strictness is not inherited transitively by a whole dependency graph.

This rule makes library boundaries important. A library can declare precise types, but callers still need to opt into strict scalar calls if they want strings such as `'10'` rejected rather than coerced. Libraries should validate their own invariants and document supported PHP versions; applications should adopt strict mode consistently rather than assuming a dependency's declaration changes their call sites.

### Return types are boundaries too

Return declarations are checked when a function returns. A strict file should not rely on an implicit scalar conversion to make a wrong return value acceptable:

```php
declare(strict_types=1);

function percentage(): int
{
    return 12.5; // TypeError in strict mode
}
```

If a function calculates a fractional result, return `float` or a value object with an explicit rounding policy. If it returns cents, calculate and store integer cents rather than hoping a scalar conversion produces the business answer.

## How It Works

At a call boundary, PHP evaluates an argument expression, checks the resulting value against the declared parameter type, and either enters the function or throws `TypeError`. Under a coercive scalar call, PHP may produce a value acceptable to the declaration. Under a strict scalar call, an inexact representation fails.

```text
evaluate argument
       ↓
check declared type
       ├── accepted value → enter function
       └── rejected value → throw TypeError
```

This is a runtime check, not a compile-time proof. Static analysis can find more errors before execution, but it cannot replace runtime checks where values arrive from HTTP, JSON, databases, queues, or extensions.

## What PHP Does

PHP's type system has grown substantially since PHP 7. Scalar parameter and return types arrived in PHP 7.0; nullable types in 7.1; typed properties in 7.4; union types in 8.0; intersection types in 8.1; DNF types and standalone `true` in 8.2; and typed class, interface, trait, and enum constants in 8.3. These features improve the vocabulary of contracts, but they do not remove the need to normalize external data.

```php
function label(int|string $id): string
{
    return (string) $id;
}

function findCourt(?int $id): ?Court
{
    return $id === null ? null : loadCourt($id);
}
```

Use a union when the domain genuinely permits alternatives. Do not use `int|string` merely because an input boundary has not been designed yet. A weak union moves ambiguity into every caller.

`mixed` means “any value” and is appropriate at a deliberate boundary, such as a generic serializer or raw message adapter. It is not a sign that a domain function has no useful contract. `never`, `void`, `object`, `iterable`, and `callable` each express different contracts and should be chosen for a reason.

## Minimal Example

```php
<?php

declare(strict_types=1);

function total(int $unitPriceCents, int $quantity): int
{
    if ($unitPriceCents < 0 || $quantity < 1) {
        throw new InvalidArgumentException('Invalid order values.');
    }

    return $unitPriceCents * $quantity;
}

echo total(1250, 3), PHP_EOL;
// total('1250', 3); // TypeError in this strict caller
```

The declaration catches representation errors. The explicit checks catch domain errors. Both are needed: an integer can be negative, and a correctly typed quantity can still exceed an inventory limit.

## Practical Example

Separate a raw request adapter from the typed application service:

```php
<?php

declare(strict_types=1);

final readonly class ReservationRequest
{
    public function __construct(
        public int $courtId,
        public int $startsAt,
        public int $endsAt,
    ) {
        if ($courtId < 1 || $startsAt >= $endsAt) {
            throw new InvalidArgumentException('Invalid reservation request.');
        }
    }
}

function makeReservation(ReservationRequest $request): void
{
    reserveCourt($request->courtId, $request->startsAt, $request->endsAt);
}

function handle(array $payload): void
{
    $request = new ReservationRequest(
        courtId: parsePositiveInt('court_id', $payload['court_id'] ?? null),
        startsAt: parsePositiveInt('starts_at', $payload['starts_at'] ?? null),
        endsAt: parsePositiveInt('ends_at', $payload['ends_at'] ?? null),
    );

    makeReservation($request);
}
```

The adapter accepts `mixed` data because that is what an external boundary actually provides. The application service has no reason to accept `mixed` or to rediscover whether a string is an integer. This is the practical relationship between Chapter 11's normalization and this chapter's declarations.

## Production Example

Adopting strict calls in an existing codebase is a migration project:

1. Add return types and parameter types where the intended contract is known.
2. Turn on strict types in a bounded module or entry point.
3. Run tests and instrument `TypeError` failures instead of catching them broadly.
4. Normalize HTTP, CLI, queue, database, and environment inputs at their adapters.
5. Fix callers that relied on implicit conversions.
6. Enable the policy for new files and expand it as ownership is clear.

Do not respond to a newly exposed type error by adding `(int)` at every call site. First decide whether the source should have supplied an integer. A cast may keep a request moving while turning malformed input into an apparently valid identifier.

## Bad Example

```php
function charge(int $amountCents): void
{
    paymentProvider()->charge($amountCents);
}

// A non-strict controller passes a request string directly.
charge($_POST['amount_cents'] ?? 0);
```

The parameter declaration looks safe, but the caller's coercive mode can convert an unexpected representation. The default `0` also collapses a missing amount into a real numeric value. Neither choice expresses the payment contract.

## Better Example

```php
declare(strict_types=1);

$rawAmount = $_POST['amount_cents'] ?? null;
$amountCents = parsePositiveInt('amount_cents', $rawAmount);

charge($amountCents);
```

The adapter validates presence and representation; strict mode prevents later callers from accidentally passing strings. The payment service can still enforce limits, currency, authorization, idempotency, and provider failure handling. Types are one layer of the design, not the whole design.

## Edge Cases

- `strict_types` is per file, not per function, class, namespace, or application.
- It primarily changes scalar argument and return coercion. It does not make untyped variables static.
- Calls made from internal functions are not changed by a userland caller's strict declaration.
- A typed property is checked when assigned. A declaration does not validate the contents of an untyped array.
- `?int` means `int|null`; it does not mean “any false-like value.”
- A union can accept a value without resolving business ambiguity. `int|string` may still be the wrong domain model for an identifier.
- `TypeError` is an `Error`, not an `Exception`. Catch it only when a boundary has a deliberate recovery or translation policy.
- `mixed` and `array` are broad contracts. Add value-level validation when the contents matter.

## Performance

Runtime type checks have a cost, but removing them to save a speculative micro-cost usually weakens the system. The dominant cost in a typical backend request is more likely to be I/O, database work, serialization, or network latency. Strong boundaries can reduce downstream defensive checks and make static analysis more effective.

Measure hot loops before changing a type boundary. If a parser processes millions of rows, normalize in batches and avoid repeated representation changes; for ordinary requests, optimize for clear failure and observability first.

## Security

Strict types reduce one class of type-confusion bug but do not provide validation or authorization. A strict `int $userId` still permits an attacker to request another user's integer ID unless policy checks ownership. A strict `string $token` still permits an invalid token.

Translate type failures at the correct outer boundary. An HTTP adapter can return a structured `400` response for malformed client input; a queue worker may reject a poison message and alert; an internal invariant failure may be a `500`. Do not catch every `TypeError` and continue with a default.

## Database Interaction

A database driver may return values in transport-specific types. Decide whether a repository returns normalized domain values or exposes driver representations. Do not let a string `'1'` versus integer `1` distinction leak into authorization logic accidentally.

Prepared statements prevent SQL syntax injection, but they do not determine whether a value is the correct domain type or whether the caller may access the row. Type normalization, parameterization, and authorization solve different problems.

## Testing

Test both sides of the type boundary:

```php
final class TotalTest extends TestCase
{
    public function testItAcceptsTypedValues(): void
    {
        self::assertSame(3750, total(1250, 3));
    }

    public function testStrictCallerRejectsNumericStrings(): void
    {
        $this->expectException(TypeError::class);
        total('1250', 3);
    }

    public function testDomainRulesRejectNegativePrices(): void
    {
        $this->expectException(InvalidArgumentException::class);
        total(-1, 3);
    }
}
```

The strictness test must be in a file with `declare(strict_types=1)`. Add integration tests for adapters so malformed JSON, form values, and environment values are rejected before the typed service is called. Static analysis should run alongside runtime tests to catch incorrect call sites earlier.

## Common Mistakes

- Believing strict types is global or transitive.
- Adding `declare(strict_types=1)` and leaving untrusted strings unvalidated.
- Using casts to silence every `TypeError` during migration.
- Treating a union type as a substitute for deciding the domain model.
- Assuming type declarations enforce ranges, formats, authorization, or database existence.
- Catching `TypeError` broadly and replacing it with a default value.
- Forgetting that a strict return type still cannot express every business invariant.

## Senior Engineer Thinking

Ask where each contract should be enforced:

1. What representation does the transport provide?
2. Which adapter owns parsing and error translation?
3. What is the narrowest useful type for the application service?
4. Which invariants need a value object rather than a scalar?
5. What migration failures reveal a real bug versus a legacy compatibility requirement?
6. Where would a type error be observable, retryable, or fatal?
7. Does the type declaration make invalid states harder to represent, or merely describe a broad container?

The mature use of strict types is not maximal annotation. It is a deliberate map of trust boundaries and domain contracts.

## Exercises

1. Create one strict caller and one coercive caller for the same `int` function. Call it with `'7'`, `7.0`, and `true`, then record the results in the PHP version you support.
2. Add strict types to a small command-line entry point. Classify each failure as missing parsing, an incorrect legacy contract, or a real bug.
3. Replace `array $payload` in a reservation service with a typed request object. Decide which checks belong in its constructor and which require a database.
4. Define a `Money` value object instead of passing `int|float|string` through a payment API. Document rounding and currency rules.

## Review Questions

1. What does `declare(strict_types=1)` change, and what does it not change?
2. Why does the caller's file matter for scalar argument coercion?
3. How do a type declaration and a domain invariant differ?
4. Why is a cast often the wrong response to a type error at an HTTP boundary?
5. When is a union type honest, and when does it preserve unresolved ambiguity?
6. Why should `TypeError` not automatically be caught and converted to a default?
7. How would you migrate a legacy codebase without turning every type failure into downtime?

## Summary

Strict types provide per-file control over scalar type coercion at function boundaries. They make representation mistakes fail earlier, but they do not make variables statically typed, validate external input, or enforce domain and authorization rules. Normalize at adapters, use narrow declarations inside the application, migrate incrementally, and test both accepted types and rejected representations. The next chapter applies this type model to PHP's operators, where precedence, comparison, short-circuiting, and assignment create equally important boundary bugs.

### Further reading

- [PHP Manual: Type declarations](https://www.php.net/manual/en/language.types.declarations.php)
- [PHP Manual: `declare`](https://www.php.net/manual/en/control-structures.declare.php)
- [PHP 7 migration guide: scalar type declarations](https://www.php.net/manual/en/migration70.new-features.php)
