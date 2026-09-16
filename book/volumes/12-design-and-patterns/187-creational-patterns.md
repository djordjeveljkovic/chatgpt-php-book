---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 187
title: Creational Patterns
slug: creational-patterns
status: complete
summary: ../../_ai/chapter-summaries/187-creational-patterns-summary.md
---

# Chapter 187 — Creational Patterns

Creational patterns control how objects and related object graphs are built. They are useful when construction has policy, validation, multiple implementations, expensive setup, or a sequence that callers should not repeat.

A pattern is not a goal. A public constructor, a named constructor, or a small factory is often clearer than a factory hierarchy. Choose the smallest creation mechanism that keeps invalid objects out and keeps callers independent of infrastructure details.

## Why this matters

Construction can be more than assigning fields. A payment client may require credentials, a sandbox mode, timeouts, and a transport. An order may need a generated identifier and validated money. If every caller assembles those details independently, behavior drifts and invalid combinations become possible.

A named constructor can express a domain rule directly:

```php
<?php

declare(strict_types=1);

final readonly class Money
{
    private function __construct(public int $cents, public string $currency)
    {
    }

    public static function fromCents(int $cents, string $currency): self
    {
        if ($cents < 0 || !preg_match('/^[A-Z]{3}$/D', $currency)) {
            throw new InvalidArgumentException('Invalid money');
        }

        return new self($cents, $currency);
    }
}
```

The private constructor prevents callers from bypassing the invariant. A named constructor is appropriate when there are a few clear creation forms.

## Factory method

A factory method chooses an implementation or performs multi-step validation behind a narrow API:

```php
interface PaymentMethod
{
    public function authorize(Money $amount): string;
}

final class CardPayment implements PaymentMethod
{
    public function __construct(private CardClient $client)
    {
    }

    public function authorize(Money $amount): string
    {
        return $this->client->authorize($amount->cents, $amount->currency);
    }
}

final class BankPayment implements PaymentMethod
{
    public function __construct(private BankClient $client)
    {
    }

    public function authorize(Money $amount): string
    {
        return $this->client->reserve($amount->cents, $amount->currency);
    }
}

final class PaymentMethodFactory
{
    public function __construct(
        private CardClient $cards,
        private BankClient $banks,
    ) {
    }

    public function create(string $kind): PaymentMethod
    {
        return match ($kind) {
            'card' => new CardPayment($this->cards),
            'bank' => new BankPayment($this->banks),
            default => throw new InvalidArgumentException('Unsupported payment method'),
        };
    }
}
```

The factory owns the mapping and rejects unknown choices. It does not need to be a singleton; inject it where creation is a real responsibility.

## Builders and complex configuration

A builder is useful when an object has many optional settings, staged validation, or a readable construction sequence. It is a poor fit when it only forwards every constructor argument with no additional policy.

```php
final class HttpClientBuilder
{
    private float $timeout = 5.0;
    private array $headers = [];

    public function timeout(float $seconds): self
    {
        if ($seconds <= 0 || $seconds > 60) {
            throw new InvalidArgumentException('Invalid timeout');
        }
        $this->timeout = $seconds;

        return $this;
    }

    public function header(string $name, string $value): self
    {
        $this->headers[$name] = $value;

        return $this;
    }

    public function build(HttpTransport $transport): HttpClient
    {
        return new HttpClient($transport, $this->timeout, $this->headers);
    }
}
```

Keep the builder mutable only during construction and return an immutable or independent product. Validate at the earliest point that has enough information, and validate cross-field rules in `build()`. Never let a builder accept secrets and then expose them through `__toString()` or debug output.

## Abstract factory and related families

An abstract factory creates related objects that must work together, such as a test or production set of storage and queue adapters:

```php
interface Storage
{
    public function put(string $key, string $contents): void;
}

interface Queue
{
    public function publish(string $type, array $payload): void;
}

interface PlatformFactory
{
    public function storage(): Storage;
    public function queue(): Queue;
}
```

A `ProductionPlatformFactory` and `InMemoryPlatformFactory` can guarantee that the selected storage and queue belong to the same environment. Do not introduce this pattern for two unrelated constructors; a composition root may be clearer.

## Singleton and global state

A singleton hides lifetime and makes tests depend on global state. It is especially risky for credentials, request data, transactions, and mutable caches. If one process-wide resource truly must be shared, construct it in the composition root and inject the shared instance. Sharing should be a deliberate lifetime decision, not a consequence of a static accessor.

Prototype-style cloning can copy configuration, but PHP object graphs may contain mutable collaborators or resources that should not be copied. Prefer an explicit factory or immutable value when clone semantics are not obvious.

## Testing and operations

Test factories with unsupported choices, invalid configuration, incompatible families, and dependency failures. Test products with their contract tests. A factory test should not assert private class names if the caller only depends on behavior; assert the capability and important configuration instead.

Creation policy belongs near configuration and should fail early. Log a safe product identity and configuration version, not credentials or full connection options. If construction opens network connections or allocates large resources, make that cost visible and bound retries and timeouts.

## Exercises

1. Replace a switch that builds payment clients in three controllers with one injected factory.
2. Decide whether a named constructor, factory method, or builder best fits a report object with two required and five optional settings.
3. Remove a singleton database client and move its shared lifetime into the composition root.

## Review questions

- When is a factory clearer than a public constructor?
- What validation belongs in a builder's setter versus `build()`?
- How does an abstract factory keep related implementations compatible?
- Why are singletons difficult to test and reason about?
- Which construction behavior should be covered by integration tests?

## Summary

Use named constructors, factories, builders, and abstract factories only when they protect invariants, select compatible implementations, or simplify meaningful construction policy. Keep creation explicit, validate early, avoid global singletons, and test products at their real boundaries.

## References

- [PHP manual: Object cloning](https://www.php.net/manual/en/language.oop5.cloning.php)
- [PHP manual: Enumerations](https://www.php.net/manual/en/language.enumerations.php)
- [Refactoring.Guru: Creational Design Patterns](https://refactoring.guru/design-patterns/creational-patterns)
- [SourceMaking: Design Patterns](https://sourcemaking.com/design_patterns)
