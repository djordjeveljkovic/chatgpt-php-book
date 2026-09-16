---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 167
title: Stubs
slug: stubs
status: complete
summary: ../../_ai/chapter-summaries/167-stubs-summary.md
---

# Chapter 167 — Stubs

A stub supplies predetermined answers to calls made by the system under test. It helps the test reach a state or branch without performing the real operation. A stub normally does not verify that the call happened or that it happened a particular number of times.

That distinction keeps a test focused. If a service should calculate shipping from a rate returned by a clock, repository, or feature provider, the test can provide that answer and assert the calculation. It should use a mock or spy only when the call itself is a requirement.

## Why this matters

External state is often variable, slow, or unavailable in a unit test. Time, exchange rates, feature flags, provider responses, and repository lookups can make a test nondeterministic. A stub turns those conditions into explicit inputs.

Define a narrow interface for the answer the application needs:

```php
<?php

declare(strict_types=1);

interface Clock
{
    public function now(): DateTimeImmutable;
}

final class DiscountPolicy
{
    public function __construct(private Clock $clock)
    {
    }

    public function discountFor(DateTimeImmutable $registeredAt): int
    {
        $age = $this->clock->now()->getTimestamp() - $registeredAt->getTimestamp();

        return $age >= 365 * 86_400 ? 15 : 0;
    }
}
```

A test can provide a fixed clock:

```php
final class FixedClock implements Clock
{
    public function __construct(private DateTimeImmutable $current)
    {
    }

    public function now(): DateTimeImmutable
    {
        return $this->current;
    }
}

public function testAnnualDiscountUsesTheConfiguredDate(): void
{
    $clock = new FixedClock(new DateTimeImmutable('2026-09-16T12:00:00+00:00'));
    $policy = new DiscountPolicy($clock);

    self::assertSame(
        15,
        $policy->discountFor(new DateTimeImmutable('2025-09-16T12:00:00+00:00')),
    );
}
```

The test controls the condition instead of sleeping or depending on the machine's current clock.

## PHPUnit stubs

PHPUnit can create a stub and configure its answer:

```php
interface CustomerRepository
{
    public function find(int $id): ?array;
}

final class GreetingService
{
    public function __construct(private CustomerRepository $customers)
    {
    }

    public function greeting(int $id): string
    {
        $customer = $this->customers->find($id);

        return $customer === null ? 'Welcome, visitor' : 'Welcome, ' . $customer['name'];
    }
}

public function testMissingCustomerUsesTheVisitorGreeting(): void
{
    $customers = $this->createStub(CustomerRepository::class);
    $customers->method('find')->willReturn(null);

    self::assertSame('Welcome, visitor', (new GreetingService($customers))->greeting(7));
}
```

The test does not claim that `find()` is called exactly once. That is intentional: the behavior under test is the greeting. If a lookup count or argument is a contract, use a mock or a spy in a separate test.

For multiple inputs, use a callback that models a small data source rather than a production algorithm:

```php
$customers->method('find')->willReturnCallback(
    static fn (int $id): ?array => $id === 7 ? ['name' => 'Mina'] : null,
);
```

A callback should remain transparent and bounded. If it grows into a database implementation, use a fake or an integration test.

## Stubbing failures and boundaries

A stub can represent a timeout, missing record, malformed upstream response, or feature flag state:

```php
$rates = $this->createStub(ExchangeRates::class);
$rates->method('rate')->willThrowException(new RuntimeException('rate service timeout'));
```

The system under test should define whether that exception is retried, converted to a domain error, or allowed to fail. The stub makes that policy testable; it does not prove that the real transport produces the exception in the same way.

Stub at the boundary where the answer matters. Stubbing a private method or replacing a value object with a dynamic mock can hide design problems. Prefer constructor injection and interfaces that express domain answers. A final concrete class may be better covered by a real instance than by forcing a mocking tool around it.

## Stubs and invalid data

A stub should be able to return values that the real boundary might produce, including empty results, duplicate records, unknown enum values, stale timestamps, and provider errors. Keep invalid fixtures deliberate and name the condition. Do not have every test use a happy-path stub; otherwise error handling becomes an untested assumption.

Do not use a stub to bypass validation that belongs in the system under test. If a JSON decoder returns an array, test the decoder separately with real input. If an adapter promises a typed `Customer`, stub the typed interface rather than an arbitrary array and let integration tests cover mapping.

## Testing and operations

Stub-based unit tests are fast and isolated, but they do not detect schema drift, network configuration errors, SQL mistakes, or differences between a real clock and a fake clock. Pair them with integration and contract tests at important boundaries. Keep a small number of high-value fixtures so changes to a provider contract are visible.

A useful review question is: “What fact does this stub provide, and which test proves the provider can provide it?” If no test answers the second part, the suite may have excellent unit coverage and a broken integration.

## Exercises

1. Add a fixed clock test for a discount boundary one second before and after the cutoff.
2. Stub a repository to return an empty collection, a duplicate ID, and a missing record. Decide which are valid domain states.
3. Add a stubbed timeout to a provider client and test the service's retry or fallback policy.

## Review questions

- What does a stub provide, and what does it intentionally avoid verifying?
- Why is a fixed clock better than sleeping in a test?
- When should a stub become a fake or an integration test?
- Why should invalid and empty answers appear in fixtures?
- Which provider facts remain untested by a stub?

## References

- [PHPUnit documentation: Test stubs](https://docs.phpunit.de/en/11.5/test-doubles.html#test-stubs)
- [PHPUnit documentation: Configuring stub behavior](https://docs.phpunit.de/en/11.5/test-doubles.html#configuring-stub-behavior)
- [Martin Fowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
