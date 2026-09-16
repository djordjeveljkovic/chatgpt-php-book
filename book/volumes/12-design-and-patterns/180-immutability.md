---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 180
title: Immutability
slug: immutability
status: complete
summary: ../../_ai/chapter-summaries/180-immutability-summary.md
---

# Chapter 180 — Immutability

## Why This Matters

An immutable value cannot change after construction. Instead of sharing an object whose state can be modified by any caller, code shares a stable value and creates a new value for an update. This reduces aliasing bugs, makes reasoning about time and concurrency easier, and improves cache and event safety.

Immutability has a cost: copying, allocation, and more explicit update code. Use it where stable values and clear ownership matter, especially value objects, commands, configuration, messages, and snapshots.

## Value Objects and Readonly

PHP supports readonly properties and readonly classes in modern versions. They prevent reassignment through the declared property, but they do not automatically make every referenced object or array deeply immutable. Model a value with scalar or immutable components where possible:

~~~php
<?php

declare(strict_types=1);

final readonly class EmailAddress
{
    public function __construct(public string $value)
    {
        $normalized = strtolower(trim($value));

        if (!filter_var($normalized, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }

        $this->value = $normalized;
    }
}
~~~

The constructor promotion above assigns before the body can normalize the promoted property, so the example would not store the normalized value. Use a normal property when construction requires normalization:

~~~php
<?php

declare(strict_types=1);

final readonly class EmailAddress
{
    public string $value;

    public function __construct(string $value)
    {
        $normalized = strtolower(trim($value));

        if (!filter_var($normalized, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }

        $this->value = $normalized;
    }
}
~~~

A value object can be safely shared because callers cannot replace its value. Its validation is performed once, and equality can be defined by value rather than object identity.

## Persistent Updates

An immutable object returns a new instance for a change:

~~~php
<?php

declare(strict_types=1);

final readonly class RetryPolicy
{
    public function __construct(
        public int $attempts,
        public int $delaySeconds,
    ) {
        if ($attempts < 1 || $delaySeconds < 0) {
            throw new InvalidArgumentException('Invalid retry policy');
        }
    }

    public function withAttempts(int $attempts): self
    {
        return new self($attempts, $this->delaySeconds);
    }
}
~~~

The old policy remains valid for a caller that still holds it. PHP's copy-on-write behavior can make copying arrays efficient until a write occurs, but it is not a reason to assume every nested object is immutable. Treat arrays containing objects as mutable unless the contained objects and ownership policy make them safe.

## Immutability and Time

Use DateTimeImmutable rather than DateTime when a timestamp should not be changed by an alias. Methods return a new instance:

~~~php
<?php

$createdAt = new DateTimeImmutable('2026-09-16T12:00:00Z');
$expiresAt = $createdAt->modify('+15 minutes');

assert($createdAt->format('c') !== $expiresAt->format('c'));
~~~

An immutable timestamp does not make a workflow immutable. A database row can still be updated, and a reservation can still transition state. Immutability applies to an object or value; domain state changes can be represented by a new aggregate version, event, or database update.

## Readonly Is Not Deep Immutability

A readonly property prevents reassignment of the property after initialization, but an object stored in it can still mutate through its own methods. A readonly array cannot have an element changed through the property, but an object nested in an array may still be mutable if it is reachable elsewhere. Encapsulate collections, return immutable snapshots, and copy data at trust boundaries when ownership is unclear.

Serialization and deserialization need care. A serialized immutable object can contain old or invalid data if the format is not validated. Treat external messages as untrusted input and reconstruct through a validating factory.

## Immutability, Concurrency, and Caching

Immutable values are easier to share between requests, workers, and fibers because readers cannot observe an in-place update. They reduce lock needs for local state, but they do not solve database races, stale caches, or duplicate side effects. Use transactions, version checks, and idempotency for shared external state.

A cache key or event payload should be built from a stable snapshot. Do not put a mutable entity in a long-lived cache and assume it remains representative of the database. Serialize immutable DTOs and include a version or timestamp when consumers need freshness semantics.

## Failure and Threat Analysis

* **Shallow readonly:** a nested object remains mutable. Use immutable components or ownership boundaries.
* **Aliasing:** two callers hold a mutable reference and one changes the other's view. Copy or encapsulate.
* **Normalization after promotion:** a readonly property stores the raw value. Assign normalized data explicitly.
* **False concurrency safety:** immutable local objects hide database races. Use transactions and versioning.
* **Allocation pressure:** copying large graphs is expensive. Use persistent data structures, snapshots, or controlled ownership.
* **Untrusted reconstitution:** serialized data bypasses constructors. Validate at the boundary.
* **Stale snapshot:** an immutable cached value is old. Define TTL, version, and invalidation policy.

## Testing Immutability

Test that updates return a new value and leave the old value unchanged. Test normalization, equality, invalid construction, nested object ownership, and serialization boundaries. Static analysis can catch attempts to reassign readonly properties, while runtime tests verify domain behavior.

Use immutable commands and events when retries or asynchronous delivery matter. The message's identity and payload should not change after publication; consumers can safely persist a copy and deduplicate by ID.

## Exercises

1. Implement an immutable DateRange with validated endpoints and methods that return new ranges.
2. Find a readonly class containing a mutable collaborator. Decide whether to copy, wrap, or change the collaborator's contract.
3. Design an immutable event DTO for an order-created message. Include schema version and event ID.
4. Compare mutable and immutable cart updates under concurrent command handling. Identify which database controls remain necessary.

## Review Questions

1. What does immutability make easier to reason about?
2. Why is readonly not automatically deep immutability?
3. How does DateTimeImmutable differ from a mutable timestamp object?
4. When should an update return a new value?
5. Why does immutability not solve database concurrency?
6. How can serialization bypass an object's invariants?

## Summary

Immutability gives callers stable values and makes aliasing, retries, and local concurrency easier to reason about. Use validated value objects, readonly properties or classes with immutable components, explicit persistent updates, immutable timestamps, and safe reconstitution. Pair local immutability with database transactions, versioning, cache freshness, and idempotency for shared state.

## References

- [PHP Manual: Readonly Classes](https://www.php.net/manual/en/language.oop5.basic.php#language.oop5.basic.class.readonly)
- [PHP Manual: Readonly Properties](https://www.php.net/manual/en/language.oop5.properties.php#language.oop5.properties.readonly)
- [PHP Manual: DateTimeImmutable](https://www.php.net/manual/en/class.datetimeimmutable.php)
- [PHP Manual: Copy-on-write](https://www.php.net/manual/en/features.gc.refcounting-basics.php)

