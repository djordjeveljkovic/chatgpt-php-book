---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 35
title: Enums
slug: enums
status: complete
summary: ../../_ai/chapter-summaries/035-enums-summary.md
---

# Chapter 35 — Enums

## Why This Matters

An untyped string such as `"pending"` carries less information than a type whose legal states are declared in one place. Enums make a finite domain explicit. They help a function accept “one of these states” instead of accepting every string and hoping validation happened earlier.

PHP enums were introduced in PHP 8.1. They are special objects: each case is a singleton instance of the enum type. A backed enum additionally maps each case to one unique `string` or `int` scalar for storage or transport.

## Mental Model

```text
external scalar ──validate/map──▶ enum case ──domain behavior──▶ result
                                  │
                                  └──persist/serialize──▶ backing value
```

Keep the enum case inside the domain. Convert at boundaries such as HTTP input, database rows, and JSON. A backing value is a representation, not a license for every layer to pass raw strings around.

## Pure and Backed Enums

```php
<?php

declare(strict_types=1);

enum ReservationStatus
{
    case Pending;
    case Confirmed;
    case Cancelled;
}

enum PaymentStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Failed = 'failed';
}
```

Pure cases have no scalar value. Backed cases must all have unique values of the enum's one backing type. There is no automatic integer numbering. `PaymentStatus::from('paid')` returns a case or throws `ValueError`; `PaymentStatus::tryFrom($value)` returns a case or `null`, making it useful for untrusted input.

## How It Works

Cases are objects and can be compared by identity:

```php
function canCancel(ReservationStatus $status): bool
{
    return $status === ReservationStatus::Pending;
}

$status = ReservationStatus::Confirmed;
assert(canCancel($status) === false);
```

Enums can have methods, constants, and interfaces. They cannot be extended, cannot be instantiated with `new`, and cannot declare instance properties. `UnitEnum::cases()` returns cases in declaration order. Backed enums expose the engine-provided `from()` and `tryFrom()` methods; do not redeclare them.

## Practical Example: Domain Transitions

Put legal transitions near the type, but keep persistence and orchestration outside it:

```php
enum ReservationStatus: string
{
    case Pending = 'pending';
    case Confirmed = 'confirmed';
    case Cancelled = 'cancelled';

    public function canTransitionTo(self $next): bool
    {
        return match ($this) {
            self::Pending => in_array($next, [self::Confirmed, self::Cancelled], true),
            self::Confirmed => $next === self::Cancelled,
            self::Cancelled => false,
        };
    }
}

function changeStatus(ReservationStatus $current, ReservationStatus $next): ReservationStatus
{
    if (!$current->canTransitionTo($next)) {
        throw new DomainException('Illegal reservation transition.');
    }

    return $next;
}
```

The `match` expression is exhaustive for the declared cases. Adding a case forces the transition policy and its tests to be reconsidered. That is a useful maintenance failure, not noise.

## Boundary Conversion

```php
function statusFromRequest(string $raw): ReservationStatus
{
    $status = ReservationStatus::tryFrom($raw);
    if ($status === null) {
        throw new InvalidArgumentException('Unknown reservation status.');
    }

    return $status;
}

$status = statusFromRequest('confirmed');
$databaseValue = $status->value;
```

Use `from()` when an invalid value indicates a broken trusted invariant, such as a corrupted migration or a programmer error. Use `tryFrom()` when the value is user-controlled and the application should choose an error response or fallback. Never silently map an unknown payment state to “paid.”

## JSON, Storage, and Serialization

Backed enums are naturally represented by their scalar backing value when JSON encoded. Pure enums need an explicit representation, often a case name or a DTO; implementing `JsonSerializable` lets the API choose a stable contract. Database columns should store the backing value with a constraint where practical, and migrations must account for adding, renaming, or removing cases.

PHP's native `serialize()` has special enum handling and restores the same singleton case. That wire format is PHP-specific and should not become an HTTP or cross-language protocol. For APIs, choose an explicit JSON schema and document whether clients receive `"paid"`, `"Paid"`, or a richer object.

## Bad Example and Better Example

Bad:

```php
function settle(string $status): void
{
    if ($status === 'paid' || $status === 'payed') {
        // Two spellings now mean one business state.
    }
}
```

Better:

```php
function settle(PaymentStatus $status): void
{
    if ($status !== PaymentStatus::Paid) {
        throw new DomainException('Payment is not settled.');
    }
}
```

The boundary still receives a string, but conversion occurs once and invalid vocabulary cannot travel through the domain API.

## Performance and Database Interaction

Enum case access and identity comparison are cheap, but the important performance question is usually conversion and query shape. Do not load every row into PHP just to filter by a backed value; let an indexed database column filter it. If the enum's values are used in a high-cardinality query, inspect the plan and index selectivity. `cases()` creates an array, so do not repeatedly build it in a hot path for a large catalog; cache only when measurement justifies it.

## Security and Compatibility

Enums prevent accidental invalid values; they do not authorize transitions. A user may submit `cancelled` and still lack permission to cancel. Do not expose internal case names as a public protocol unless you intend to support them. Renaming a case can break PHP code and native serialized payloads even if the backing value remains unchanged. Prefer stable backing values for persisted data and plan migrations for semantic changes.

## Testing

Test the finite domain, conversion, and transition matrix:

```php
assert(PaymentStatus::from('paid') === PaymentStatus::Paid);
assert(PaymentStatus::tryFrom('unknown') === null);
assert(count(ReservationStatus::cases()) === 3);

try {
    ReservationStatus::from('paid'); // no from() on a pure enum
    assert(false);
} catch (Error $error) {
    assert(true);
}
```

For a transition enum, use a data provider containing every current-to-next combination, including self-transitions and unknown boundary values. Add a database integration test proving that the stored scalar round-trips to the expected case.

## Common Mistakes

- Treating enums as glorified constants while passing raw strings everywhere.
- Assuming backed enum values are automatically generated or globally unique.
- Using `from()` directly on arbitrary request input without handling `ValueError`.
- Letting JSON representation accidentally become a long-term API contract.
- Forgetting that adding a case requires revisiting exhaustive `match` expressions and transition rules.

## Senior Engineer Thinking

An enum is a small domain model, not only a list. Give it behavior when that behavior is intrinsic to the state vocabulary. Keep workflow decisions, authorization, persistence, and external calls in services. Choose pure versus backed based on whether a stable scalar boundary exists, and make the storage/API contract deliberate.

## Exercises

1. Model reservation status transitions as an enum and write the complete transition table.
2. Add an enum for court surface and decide whether the database should store its case name or a stable backing value.
3. Design a JSON representation for a pure enum that can evolve without exposing PHP class names.

## Review Questions

1. How do pure and backed enums differ?
2. When should `from()` be preferred over `tryFrom()`?
3. Why are enum cases objects rather than strings?
4. What can break when an enum case is renamed?
5. Why does an enum not replace authorization?

## Summary

Enums make finite domain states explicit and type-checkable. Use cases inside the domain, convert at boundaries, choose backed values deliberately for storage, and test transitions exhaustively. Their correctness benefit comes from removing invalid vocabulary, not from eliminating the need for persistence and security design.

## Official References

- [PHP Manual: Enumerations overview](https://www.php.net/manual/en/language.enumerations.overview.php)
- [PHP Manual: Basic enumerations](https://www.php.net/manual/en/language.enumerations.basics.php)
- [PHP Manual: Backed enumerations](https://www.php.net/manual/en/language.enumerations.backed.php)
- [PHP Manual: Enumeration methods](https://www.php.net/manual/en/language.enumerations.methods.php)
