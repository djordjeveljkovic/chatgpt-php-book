---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 37
title: Magic Methods
slug: magic-methods
status: complete
summary: ../../_ai/chapter-summaries/037-magic-methods-summary.md
---

# Chapter 37 — Magic Methods

## Why This Matters

PHP invokes certain methods when ordinary syntax encounters an object: construction, cloning, string conversion, missing properties, serialization, and debugging. These methods are “magic” only in the sense that the engine recognizes reserved names. They are hidden control flow, and hidden control flow is easy to misuse.

Use a magic method when the object protocol genuinely requires it. Prefer explicit methods for business operations. A method such as `toArray()` tells a reader what is happening; `__get()` makes an apparently simple property read run arbitrary code.

## Mental Model

```text
ordinary syntax ──▶ engine checks object protocol ──▶ magic hook, if declared
```

Magic methods are object lifecycle and interoperability hooks, not a general metaprogramming replacement. Names beginning with `__` are reserved by PHP. Most magic methods must be public and must use the exact documented signature; `__construct()`, `__destruct()`, and `__clone()` are the visibility exceptions.

## The Common Hooks

| Hook | Trigger | Typical responsibility |
| --- | --- | --- |
| `__construct` | `new` | Establish a valid object |
| `__destruct` | object destruction | Release or finalize best-effort resources |
| `__get`, `__set` | inaccessible property access | Compatibility or controlled lazy access |
| `__isset`, `__unset` | `isset`/`unset` on inaccessible property | Match virtual property behavior |
| `__call`, `__callStatic` | inaccessible method call | Narrow proxy/decorator behavior |
| `__toString` | string context | Safe human-readable representation |
| `__invoke` | object called as function | Callable object protocol |
| `__clone` | `clone` | Repair copied object graph |
| `__serialize`, `__unserialize` | native serialization | Explicit persistence representation |
| `__debugInfo` | `var_dump` | Safe diagnostic projection |

## Minimal Example

```php
<?php

declare(strict_types=1);

final class ReservationLabel
{
    public function __construct(
        private string $court,
        private string $date,
    ) {
    }

    public function __toString(): string
    {
        return $this->court . ' on ' . $this->date;
    }
}

echo new ReservationLabel('Court 1', '2026-09-14');
```

`__toString()` is appropriate because string conversion is the object's natural textual representation. It must not leak secrets or perform a database query merely because a logger interpolated the object.

## Property Overloading: `__get`, `__set`, `__isset`, `__unset`

These hooks run only for inaccessible properties in the relevant scope, not as a universal interception layer:

```php
final class Attributes
{
    /** @param array<string, mixed> $values */
    public function __construct(private array $values) {}

    public function __get(string $name): mixed
    {
        return $this->values[$name] ?? null;
    }

    public function __isset(string $name): bool
    {
        return isset($this->values[$name]);
    }
}
```

This can be useful at a legacy data boundary, but it weakens static analysis and makes typos look like missing data. A typed `get(string $name): mixed` or a dedicated DTO is often clearer. If `__get()` lazily loads from a database, property access has hidden I/O, latency, failure, and N+1-query implications.

## Method Overloading and Callable Objects

`__call()` can implement a narrow proxy:

```php
final class LoggingGateway
{
    public function __construct(private object $gateway) {}

    public function __call(string $method, array $arguments): mixed
    {
        // Validate an allow-list before forwarding in real code.
        return $this->gateway->{$method}(...$arguments);
    }
}
```

Blind forwarding is dangerous. It can expose methods unintentionally, obscure stack traces, and turn user-controlled method names into a capability. Prefer explicit delegation or an allow-list. `__invoke()` is more predictable for a value that conceptually is a callable, such as a parser or predicate.

## Serialization and Debugging Hooks

`__serialize()` and `__unserialize()` define an explicit array representation. They are covered fully in Chapter 39. `__debugInfo()` controls what `var_dump()` displays:

```php
final class ApiCredential
{
    public function __construct(private string $token) {}

    public function __debugInfo(): array
    {
        return ['token' => '[redacted]'];
    }
}
```

Debug projections are not an access-control mechanism. A developer can still use reflection or inspect a process; the hook simply prevents routine dumps and logs from exposing the token.

## What PHP Does and Runtime Cost

The engine resolves ordinary property or method access, checks visibility and existence, and invokes a hook when the protocol permits it. The hook can throw, recurse, perform I/O, or mutate state. That means a one-line expression can have significant runtime cost. Magic dispatch is not itself a performance strategy.

## Bad Example and Better Example

Bad:

```php
echo $reservation->customer->address->city;
```

where every object uses `__get()` to lazy-load the next relation. The line hides several queries and failure points. Better code makes loading explicit:

```php
$customer = $customerRepository->get($reservation->customerId);
$address = $addressRepository->getForCustomer($customer->id);
echo $address->city;
```

An ORM may deliberately use magic methods, but then query count, eager loading, and failure behavior must be visible in documentation and tests.

## Edge Cases

- `__toString()` must return a string. Since PHP 7.4 it may throw; do not rely on exceptions in older supported versions.
- Since PHP 8.0, a class with `__toString()` implicitly implements `Stringable`.
- Magic methods other than the constructor, destructor, and clone hook should be public with documented signatures.
- `__debugInfo()` should not include passwords, tokens, or personal data. PHP 8.5 deprecates returning `null`; return an array, including an empty array.
- `__sleep()`/`__wakeup()` are legacy serialization hooks; new code should use `__serialize()`/`__unserialize()`.

## Security, Concurrency, and Testing

Magic methods expand the attack and failure surface. Treat dynamic names and values as untrusted. Do not let `__wakeup()` or `__unserialize()` perform privileged actions based on unchecked data. In concurrent web requests, each request has its own object instance, but a long-running worker can retain mutated state after a magic hook runs.

Test triggers explicitly: string conversion does not leak secrets, missing properties follow the declared policy, inaccessible calls are rejected or forwarded only to allowed methods, and debug output is redacted. Add integration tests for lazy loading to assert query count and transaction behavior.

## Common Mistakes

- Calling all hidden behavior “magic” and accepting it without a contract.
- Using `__get()` as a replacement for typed properties.
- Forwarding arbitrary methods through `__call()`.
- Performing database access or network calls during string conversion or debugging.
- Forgetting exact signatures and visibility requirements.

## Senior Engineer Thinking

Ask whether the syntax is genuinely part of the object's protocol. If a reader needs to know that code can throw, query, authorize, or allocate, an explicit method is usually better. Magic hooks are strongest at lifecycle boundaries and interoperability; they are weakest as an invisible business API.

## Exercises

1. Implement `__debugInfo()` for an object containing a secret and verify that routine dumps redact it.
2. Replace a dynamic `__get()` model with an explicit typed projection. Record what static analysis and query observability improve.
3. Build a safe proxy with an allow-list for two methods and test that an unknown method is rejected.

## Review Questions

1. When is `__get()` invoked?
2. Why can magic property access cause N+1 queries?
3. What security problem exists in unrestricted `__call()` forwarding?
4. What is `__debugInfo()` for, and what is it not?
5. Why should business operations usually be explicit methods?

## Summary

Magic methods are engine-recognized object protocol hooks. They can make lifecycle and interoperability work cleanly, but they hide control flow and may introduce I/O, exceptions, security exposure, and performance cost. Use narrow hooks, exact signatures, explicit tests, and explicit methods for business behavior.

## Official References

- [PHP Manual: Magic Methods](https://www.php.net/manual/en/language.oop5.magic.php)
- [PHP Manual: Object Cloning](https://www.php.net/manual/en/language.oop5.cloning.php)
- [PHP Manual: Object Serialization](https://www.php.net/manual/en/language.oop5.serialization.php)
