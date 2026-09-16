---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 202
title: Domain Services
slug: domain-services
status: complete
summary: ../../_ai/chapter-summaries/202-domain-services-summary.md
---

# Chapter 202 — Domain Services

A domain service holds domain logic that is meaningful but does not naturally belong to one entity or value object. It represents an operation in the domain language, usually has no identity, and should be stateless apart from explicitly supplied collaborators.

A domain service is not a synonym for “code that did not fit in a controller.” Its contract should express a domain decision or invariant. Application coordination, transactions, HTTP, persistence, and retries belong at other boundaries.

## Why this matters

Some rules involve two or more aggregates. A transfer compares balances and currency rules across accounts. A pricing policy combines a product, customer segment, and date. Putting the rule in a controller duplicates it across entry points; putting it in one aggregate can give that aggregate knowledge it does not own.

A domain service can name the rule:

```php
<?php

declare(strict_types=1);

final readonly class Money
{
    public function __construct(public int $cents, public string $currency)
    {
        if ($cents < 0 || !preg_match('/^[A-Z]{3}$/D', $currency)) {
            throw new InvalidArgumentException('Invalid money');
        }
    }
}

final readonly class Customer
{
    public function __construct(public int $id, public string $segment)
    {
    }
}

final class PricingService
{
    public function price(Money $base, Customer $customer, DateTimeImmutable $date): Money
    {
        $discount = match ($customer->segment) {
            'preferred' => 10,
            'staff' => 25,
            default => 0,
        };

        // The rule is deliberately deterministic and has no database knowledge.
        return new Money((int) round($base->cents * (100 - $discount) / 100), $base->currency);
    }
}
```

The class is a domain service because the discount is a domain policy involving values supplied by the caller. The date parameter is currently unused; remove it or use it for a real effective-date rule rather than carrying speculative context.

## Entity behavior first

Do not extract every method into a service. An entity should protect invariants about its own state:

```php
final class Account
{
    public function __construct(private int $balanceCents)
    {
    }

    public function debit(int $amountCents): void
    {
        if ($amountCents < 1 || $amountCents > $this->balanceCents) {
            throw new DomainException('Insufficient balance');
        }
        $this->balanceCents -= $amountCents;
    }

    public function credit(int $amountCents): void
    {
        if ($amountCents < 1) {
            throw new InvalidArgumentException('Amount must be positive');
        }
        $this->balanceCents += $amountCents;
    }
}
```

A transfer service can coordinate two account decisions without exposing balance mutation to every caller:

```php
final class TransferService
{
    public function transfer(Account $from, Account $to, int $amountCents): void
    {
        $from->debit($amountCents);
        $to->credit($amountCents);
    }
}
```

The service does not provide atomicity. The application service and repository transaction boundary must persist both changes together, and the database may need locking or a conditional update. Domain purity and persistence consistency are related but different concerns.

## Domain service versus application service

A domain service answers “what is allowed or what is the domain result?” An application service answers “which use case steps happen, in what transaction, for which actor, and through which ports?”

A domain service should not read `$_POST`, return an HTTP response, begin a PDO transaction, or dispatch a queue message. It may depend on a domain port when the domain decision requires a capability, but the port should be expressed in domain terms and its consistency assumptions documented.

Keep domain services small and cohesive. A `DomainService` class with dozens of unrelated methods is a service bag, not a design. Split policies by language and invariant.

## Time, randomness, and external facts

Domain logic becomes easier to test when time, randomness, and external facts arrive as explicit values or narrow ports:

```php
interface ExchangeRate
{
    public function rate(string $from, string $to): string;
}

final class CurrencyConversion
{
    public function __construct(private ExchangeRate $rates)
    {
    }

    public function convert(Money $amount, string $target): Money
    {
        $rate = $this->rates->rate($amount->currency, $target);
        $cents = (int) round($amount->cents * (float) $rate);

        return new Money($cents, $target);
    }
}
```

The port does not make the rate fresh or transactional. The application must define how stale rates, provider failure, and retries affect the use case. If the rule is pure after a rate is supplied, test the pure decision separately.

## Testing and operations

Use unit tests for boundary values, currencies, segments, effective dates, and invalid transitions. Test domain services with real value objects and entities; mocks should be limited to domain ports whose interaction is itself a contract. Test the persistence and transaction behavior at the repository/application boundary.

Name metrics around decisions and failures, not internal method calls. If a domain rule depends on a remote fact, observe freshness and failure at its adapter. Keep domain error types safe to map into API responses without exposing infrastructure details.

## Exercises

1. Extract a tax calculation that currently lives in a controller into a stateless domain service and value objects.
2. Decide whether a reservation conflict belongs on an aggregate, in a domain service, or in the database constraint. Explain the invariant and concurrency boundary.
3. Add an explicit clock or effective date to a promotion policy and test the boundary dates.

## Review questions

- When is a domain service preferable to an entity method?
- Which responsibilities belong to an application service instead?
- Why does a domain service not provide transaction atomicity by itself?
- How should time and external facts enter domain logic?
- What makes a domain service a cohesive policy instead of a service bag?

## Summary

Use domain services for cohesive domain decisions that span entities or do not have a natural owner. Keep them expressed in domain language, deterministic where possible, independent of HTTP and persistence, and paired with application and database boundaries that provide authorization and consistency.

## References

- [Martin Fowler: Domain Model](https://martinfowler.com/eaaCatalog/domainModel.html)
- [Martin Fowler: Service Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [PHP manual: Enumerations](https://www.php.net/manual/en/language.enumerations.php)
