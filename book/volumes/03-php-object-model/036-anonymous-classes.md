---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 36
title: Anonymous Classes
slug: anonymous-classes
status: complete
summary: ../../_ai/chapter-summaries/036-anonymous-classes-summary.md
---

# Chapter 36 — Anonymous Classes

## Why This Matters

Sometimes a dependency needs one small implementation: a clock for one test, a callback object for one adapter, or a visitor that is used once. Creating a named class may add useful clarity—or it may create a permanent name for behavior that has no reuse or identity outside its call site. Anonymous classes provide a real object without a user-chosen class name.

They are a tool for local scope, not a replacement for every small class. The trade-off is discoverability: a named class is easier to document, reuse, instrument, and reference in configuration.

## Mental Model

```text
new class(...) implements Contract { ... }
        │
        └── an ordinary object with an engine-generated class name
```

An anonymous class can have a constructor, extend a class, implement interfaces, use traits, and—in PHP 8.3 and later—be declared readonly. Every object created by the same anonymous class declaration has the same generated class identity. The generated name is an implementation detail and must not be persisted or compared as a stable string.

## Minimal Example

```php
<?php

declare(strict_types=1);

interface Clock
{
    public function now(): DateTimeImmutable;
}

$fixedClock = new class implements Clock
{
    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable('2026-09-14 12:00:00 UTC');
    }
};

function createdAt(Clock $clock): DateTimeImmutable
{
    return $clock->now();
}

assert(createdAt($fixedClock)->format('c') === '2026-09-14T12:00:00+00:00');
```

The consumer sees the stable `Clock` contract, not the anonymous class's name. This is a useful test double when the behavior is genuinely local.

## How It Works

The `new class` expression creates an object from a class declaration embedded in the expression. Constructor arguments follow `class`, and `extends`, `implements`, and `use` work as they do for named classes:

```php
interface ReservationRepository
{
    public function save(Reservation $reservation): void;
}

$repository = new class implements ReservationRepository
{
    /** @var list<Reservation> */
    public array $saved = [];

    public function save(Reservation $reservation): void
    {
        $this->saved[] = $reservation;
    }
};
```

The object is type-compatible with the interface. Its class is not a convenient public type for a function signature, so type the consumer against the interface or parent class.

## Capturing Dependencies

An anonymous class does not receive lexical variables as a closure does. Pass dependencies explicitly through the constructor:

```php
$prefix = '[reservation]';

$logger = new class($prefix)
{
    public function __construct(private string $prefix) {}

    public function line(string $message): string
    {
        return $this->prefix . ' ' . $message;
    }
};

echo $logger->line('created');
```

This makes ownership and lifetime visible. The anonymous class nested in another class also does not automatically gain access to the outer object's private or protected members. Pass the required value or dependency; do not assume lexical privilege.

## Practical Example: One-Off Policy

```php
interface RetryPolicy
{
    public function delaySeconds(int $attempt): int;
}

function runWithRetry(RetryPolicy $policy, callable $operation): mixed
{
    for ($attempt = 1; $attempt <= 3; $attempt++) {
        try {
            return $operation();
        } catch (RuntimeException $error) {
            if ($attempt === 3) {
                throw $error;
            }
            sleep($policy->delaySeconds($attempt));
        }
    }

    throw new LogicException('Unreachable.');
}

$policy = new class implements RetryPolicy
{
    public function delaySeconds(int $attempt): int
    {
        return $attempt;
    }
};
```

The example is readable at the call site, but if retry policy becomes configurable, shared, or operationally important, promote it to a named class. Anonymous code should not hide a policy that needs a dashboard, documentation, or independent tests.

## Readonly Anonymous Classes

PHP 8.3 permits:

```php
$requestContext = new readonly class('request-123')
{
    public function __construct(public string $requestId) {}
};
```

This is useful for a tiny immutable test or adapter value. The same readonly rules apply as for named classes: declared properties must be typed, static and dynamic properties are not available, and readonly does not make an object held by a property deeply immutable.

## Bad Example and Better Example

Bad code makes an anonymous class a hidden global service and later identifies it by `get_class()`:

```php
$handler = new class implements Handler { /* ... */ };
registerHandler(get_class($handler), $handler); // unstable name
```

Better code registers the behavior through a stable contract or explicit key:

```php
registerHandler('reservation-created', $handler);
```

Do not persist, log as an identity, or deserialize engine-generated anonymous class names. They can vary with source location, file, or runtime details.

## Performance, Security, and Operations

The declaration is compiled with the surrounding file, while each `new class` expression still creates an object. Repeated construction has ordinary allocation cost. If the object is stateless, a named singleton-like value or a closure may be simpler; do not optimize before measuring.

Anonymous classes do not sandbox code. They can access any dependency passed to them and can perform all the side effects of a named class. In long-running workers, avoid capturing large objects accidentally and make lifecycle ownership explicit. A test double that keeps a reference to a container or database connection can retain far more memory than expected.

## Testing

Test the contract, not the generated class name:

```php
assert($fixedClock instanceof Clock);
assert($repository instanceof ReservationRepository);

$sameDeclaration = new class implements Clock
{
    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable('@0');
    }
};

// Do not assert get_class($fixedClock) equals a hard-coded engine name.
assert($sameDeclaration instanceof Clock);
```

Anonymous classes are convenient for small fakes, but a named fake is usually better when it has multiple scenarios, failure controls, or reuse across tests. Static analysis and interface checks catch drift in the contract.

## Common Mistakes

- Treating the generated class name as stable.
- Assuming an anonymous class captures variables like a closure.
- Hiding a central business policy inside a large expression.
- Testing implementation details instead of the interface contract.
- Keeping accidental references to heavyweight services in worker processes.

## Senior Engineer Thinking

Ask whether the class has a meaningful name in the domain or architecture. If yes, name it. If the object is a local implementation of a stable contract and its behavior is short, an anonymous class can reduce ceremony. The choice is about the cost of future discovery and change, not line count alone.

## Exercises

1. Create an anonymous `Clock` for a deterministic reservation test, then refactor it to a named fake when you add time advancement.
2. Implement a one-off `ReservationRepository` that records calls and assert it through the interface.
3. Find a place where an anonymous class would hide a production policy. Explain why a named class is the safer seam.

## Review Questions

1. What capabilities does an anonymous class have compared with a named class?
2. Why should generated class names not be persisted?
3. How are dependencies passed to an anonymous class?
4. When should a one-off implementation become a named class?
5. What is the difference between an anonymous class and a closure in variable capture?

## Summary

Anonymous classes create ordinary objects for local implementations. Type consumers against stable interfaces, pass dependencies explicitly, and promote behavior to a named class when reuse, observability, configuration, or domain meaning grows. The engine-generated name is not an application identity.

## Official References

- [PHP Manual: Anonymous classes](https://www.php.net/manual/en/language.oop5.anonymous.php)
- [PHP Manual: Classes and Objects](https://www.php.net/manual/en/language.oop5.php)
