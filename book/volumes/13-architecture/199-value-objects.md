---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 199
title: Value Objects
slug: value-objects
status: complete
summary: ../../_ai/chapter-summaries/199-value-objects-summary.md
---

# Chapter 199 — Value Objects

## Why This Matters

A value object represents a concept defined by its values rather than by a separate identity. An email address, money amount, date range, country code, or percentage can validate itself and make illegal states harder to construct.

Value objects reduce primitive obsession. A string called email, a string called currency, and a string called status can all be passed through the same function signature while obeying different rules. A typed value object communicates intent and centralizes equality, formatting, and domain validation.

## Immutability and Equality

Value objects are usually immutable. An operation that appears to change a value returns a new object, leaving the original safe to share. Equality compares normalized values and relevant units, not object references.

~~~php
<?php

declare(strict_types=1);

final readonly class EmailAddress
{
    public readonly string $value;

    public function __construct(string $value)
    {
        $normalized = strtolower(trim($value));
        if (!filter_var($normalized, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email address');
        }

        $this->value = $normalized;
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }
}
~~~

Normalization is a domain decision. Lowercasing the entire address is common for account identifiers but may not match every mail system's local-part semantics. Choose and document the policy rather than assuming that every string comparison should be case-insensitive.

## Money Without Floating Point

Represent money in the smallest currency unit with an explicit currency. Do not use binary floating-point values for financial equality or rounding. The rounding mode and minor-unit rules belong to the domain or a currency policy.

~~~php
<?php

declare(strict_types=1);

final readonly class Money
{
    public function __construct(
        public int $minorUnits,
        public string $currency,
    ) {
        if ($minorUnits < 0 || !preg_match('/^[A-Z]{3}$/D', $currency)) {
            throw new InvalidArgumentException('Invalid money');
        }
    }

    public function add(self $other): self
    {
        $this->sameCurrency($other);

        return new self($this->minorUnits + $other->minorUnits, $this->currency);
    }

    public function allocate(int $parts): array
    {
        if ($parts < 1) {
            throw new InvalidArgumentException('Parts must be positive');
        }

        $base = intdiv($this->minorUnits, $parts);
        $remainder = $this->minorUnits % $parts;
        $result = [];
        for ($i = 0; $i < $parts; ++$i) {
            $result[] = new self(
                $base + ($i < $remainder ? 1 : 0),
                $this->currency,
            );
        }

        return $result;
    }

    private function sameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new DomainException('Currencies differ');
        }
    }
}
~~~

Allocation makes the remainder rule explicit. For currencies with non-two-digit minor units or cash rounding, use a currency table or money library rather than assuming every currency behaves like EUR cents.

## Date Ranges and Invariants

A DateRange can define inclusive or half-open boundaries, timezone policy, overlap, and duration. Put the convention in the type so every caller does not implement a different comparison.

~~~php
<?php

declare(strict_types=1);

final readonly class DateRange
{
    public function __construct(
        public DateTimeImmutable $start,
        public DateTimeImmutable $end,
    ) {
        if ($end <= $start) {
            throw new InvalidArgumentException('End must be after start');
        }
    }

    public function overlaps(self $other): bool
    {
        return $this->start < $other->end && $other->start < $this->end;
    }

    public function contains(DateTimeImmutable $instant): bool
    {
        return $this->start <= $instant && $instant < $this->end;
    }
}
~~~

The half-open interval makes adjacent ranges non-overlapping and avoids double-counting a boundary. State whether values are normalized to UTC or retain a business timezone. A value object cannot decide ambiguous daylight-saving times without a documented policy.

## Serialization and Persistence

Map value objects explicitly to database columns, JSON, and messages. A Money value may use two columns, a structured object, or a provider representation. Store enough precision and unit information to reconstitute the same value. Do not serialize PHP objects into a long-lived protocol unless the format and compatibility policy are explicit.

A value object's constructor may be strict while historical data requires migration. Treat invalid stored data as a migration or operational error; do not silently create an invalid object or replace it with a default. API validation can produce client-friendly errors before the domain constructor enforces the invariant.

## Value Objects and Collections

A collection can be a value object when it has value semantics and invariants such as uniqueness, ordering, or a maximum size. Keep iteration and transformation methods explicit. Do not make every array a wrapper if it adds no rule or useful vocabulary.

Nested value objects can make an aggregate expressive: Reservation contains ReservationId, EmailAddress, and DateRange. The aggregate still coordinates identity and state transitions; value objects should not reach into repositories or send network requests.

## Testing and Performance

Test construction boundaries, normalization, equality, arithmetic, invalid units, timezone behavior, serialization round trips, and edge cases. Property-based tests are useful for money allocation because the sum of parts should equal the original and each part should differ by at most one unit.

Immutable objects can be shared safely, but they still consume memory. Avoid creating millions of wrapper objects in a hot import path without measuring. A mapper can validate at the boundary and use a compact representation for batch processing when the domain does not need object behavior for every row.

## Common Mistakes

- Using floats for money.
- Treating all string normalization as universally safe.
- Leaving timezone and interval boundary rules implicit.
- Letting value objects perform I/O.
- Serializing PHP object internals into public contracts.
- Returning default values for invalid persisted data.
- Wrapping primitives without adding validation, semantics, or invariants.

## Senior Engineer Thinking

A value object earns its place by making a concept's equality, validation, normalization, and operations explicit. Keep it immutable where possible, map it deliberately at persistence boundaries, encode units and timezone rules, and measure allocation cost in high-volume paths.

## Exercises

1. Implement a CountryCode value object with normalization and an allow-list policy.
2. Add tests proving money allocation preserves the total and handles remainders deterministically.
3. Define whether a scheduling range is half-open or closed and test adjacent and overlapping intervals.
4. Design a database and JSON mapping for Money that preserves currency and minor units.

## Review Questions

1. How does value equality differ from entity identity?
2. Why are money values usually stored as integer minor units?
3. What interval convention does DateRange use, and why?
4. Which normalization rules require domain documentation?
5. Why should value objects avoid network or database access?

## Summary

Value objects make domain concepts explicit through validation, immutable values, equality, and operations. Use them for money, identifiers, ranges, and other concepts with units or invariants; avoid floating point for monetary state, document normalization and timezone rules, map values explicitly across boundaries, and test edge cases and algebraic properties.

## References

- [Martin Fowler: Value Object](https://martinfowler.com/bliki/ValueObject.html)
- [PHP readonly classes and properties](https://www.php.net/manual/en/language.oop5.basic.php)
- [PHP DateTimeImmutable](https://www.php.net/manual/en/class.datetimeimmutable.php)
- [ISO 4217 currency codes](https://www.iso.org/iso-4217-currency-codes.html)
