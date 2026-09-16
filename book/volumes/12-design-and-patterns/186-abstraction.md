---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 186
title: Abstraction
slug: abstraction
status: complete
summary: ../../_ai/chapter-summaries/186-abstraction-summary.md
---

# Chapter 186 — Abstraction

An abstraction names the behavior a caller needs while hiding details that should be allowed to change. A useful abstraction reduces the number of decisions a caller must make. A weak abstraction merely renames a concrete class, leaks implementation details, or predicts changes that never arrive.

Abstraction is a design choice with a cost. Every interface, base class, and wrapper adds a vocabulary, indirection, documentation, and test surface. Introduce one when it protects a boundary or gives a domain concept a stable meaning.

## Why this matters

A controller that knows HTTP details, SQL syntax, provider payloads, and retry rules is coupled to every change in those systems. An abstraction can give it one application-level operation:

```php
interface FraudChecker
{
    public function check(Payment $payment): FraudDecision;
}
```

The controller does not need to know whether the implementation calls a provider, uses a local rule engine, or returns a cached decision. It still needs a clear contract: possible decisions, failure behavior, timeout policy, and whether the operation is safe to retry.

## Find the stable concept

Start with a requirement and its invariants, not with a desire to add an interface. “Send an invoice” may be a useful port if the application needs delivery semantics. `InvoiceServiceInterface` with one method that mirrors a concrete service often adds little.

A good abstraction has:

- a name from the domain or integration boundary;
- a small set of operations that callers actually need;
- explicit input and output types;
- defined failure and consistency behavior;
- no accidental leakage of SQL, HTTP, framework, or vendor types.

```php
<?php

declare(strict_types=1);

enum FraudDecision: string
{
    case Approve = 'approve';
    case Review = 'review';
    case Reject = 'reject';
}

final readonly class Payment
{
    public function __construct(public int $amountCents, public string $country)
    {
    }
}

interface FraudChecker
{
    public function check(Payment $payment): FraudDecision;
}
```

The return type captures a finite decision. The interface does not expose the provider's score, HTTP response, or internal model unless those are part of the application contract.

## Leaky abstractions

An abstraction leaks when callers must understand the hidden mechanism to use it safely. A repository that returns a PDO statement, requires SQL fragments, or exposes transaction flags is not hiding the database boundary. A “generic” HTTP wrapper that accepts every cURL option transfers transport complexity to every caller.

Leaking details creates two sources of truth. If callers must know that a provider uses eventual consistency, the provider policy belongs in the abstraction's documentation or in a domain operation that reflects it. If every caller must inspect a vendor exception, map it at the adapter boundary.

Do not hide important cost or failure. An abstraction called `getExchangeRate()` that performs a network request needs timeout, caching, and failure semantics visible to its callers or represented by a result type. Hiding syntax is useful; hiding operational behavior is dangerous.

## Interfaces, abstract classes, and values

Use an interface for a capability that can have unrelated implementations. Use an abstract class when implementations share real invariant-preserving behavior and state. Use a value object when the concept is data with rules, such as `Money`, `EmailAddress`, or `DateRange`.

```php
final readonly class Money
{
    public function __construct(public int $cents, public string $currency)
    {
        if ($cents < 0 || !preg_match('/^[A-Z]{3}$/D', $currency)) {
            throw new InvalidArgumentException('Invalid money value');
        }
    }

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new DomainException('Currencies differ');
        }

        return new self($this->cents + $other->cents, $this->currency);
    }
}
```

The value object is an abstraction with behavior, not a bag of public fields that every caller validates independently. Its invariant is established once and reused.

## Abstraction at boundaries

Ports and adapters are useful when application policy must remain independent of an external mechanism:

```php
interface PaymentAuthorizer
{
    public function authorize(Payment $payment, string $requestKey): AuthorizationResult;
}

final class ProviderPaymentAuthorizer implements PaymentAuthorizer
{
    public function __construct(private ProviderClient $client)
    {
    }

    public function authorize(Payment $payment, string $requestKey): AuthorizationResult
    {
        $response = $this->client->authorize(
            $payment->amountCents,
            $payment->country,
            $requestKey,
        );

        return AuthorizationResult::fromProviderResponse($response);
    }
}
```

The adapter translates. It should not force the domain to understand provider status codes or nested JSON. Contract tests can verify the adapter; unit tests can use a narrow fake or stub for `PaymentAuthorizer`.

## Costs and evolution

An abstraction that has one implementation can still be justified when it marks a critical boundary, but do not add interfaces solely to satisfy a pattern checklist or make every class mockable. Start with a concrete implementation when no alternate behavior or boundary exists, then extract when the pressure becomes visible.

Watch for “god interfaces” with unrelated methods, interfaces named after implementations, and abstractions that require callers to pass an options array containing every hidden detail. Split capabilities or redesign the operation around the caller's actual need. A breaking change to a well-owned abstraction should be deliberate; a leaked vendor type makes every vendor upgrade a broad change.

## Testing and operations

Test the abstraction's invariants at the level where they are defined. Test each adapter against the external contract and test the application with a deterministic implementation. Test timeouts, unknown provider responses, partial failures, and retry behavior when they affect the contract.

Measure real cost at the adapter boundary. An abstraction must not make a database query or network call appear free to callers. Include latency, failure rate, and resource use in operational documentation and instrumentation.

## Exercises

1. Find an interface that exposes a vendor response type. Replace it with an application result and map the vendor response in an adapter.
2. Design a `Money` value object with currency and non-negative amount invariants.
3. Review a broad `Storage` interface and split it into capabilities used by a downloader and an uploader.

## Review questions

- What makes an abstraction useful rather than a renamed concrete class?
- Which details should an adapter translate, and which failures must remain visible?
- When is a value object a better abstraction than an interface?
- Why are broad options arrays often a leaky abstraction?
- How can an abstraction hide syntax without hiding operational cost?

## Summary

Use abstractions to protect stable concepts and boundaries. Keep contracts small, typed, and explicit about failure and cost; translate vendor details at adapters; use value objects for invariants; and accept the indirection cost only when it reduces meaningful change.

## References

- [PHP manual: Interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)
- [PHP manual: Enumerations](https://www.php.net/manual/en/language.enumerations.php)
- [Martin Fowler: Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
