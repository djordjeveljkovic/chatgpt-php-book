---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 159
title: Unit Tests
slug: unit-tests
status: complete
summary: ../../_ai/chapter-summaries/159-unit-tests-summary.md
---

# Chapter 159 — Unit Tests

## Why This Matters

A unit test checks a small unit of behavior in isolation from slow or unreliable boundaries. The unit may be a value object, function, policy, parser, or domain service. Fast unit tests let a developer change code and receive feedback in seconds, which makes experimentation and debugging practical.

Isolation is about controlling collaborators and state, not about making every class tiny or mocking every call. A unit should still express useful behavior and preserve the invariants that belong to it.

## A Small Unit and Its Contract

Choose an input boundary and an observable result. A value object can enforce its own invariant:

~~~php
<?php

declare(strict_types=1);

final readonly class Percentage
{
    public function __construct(public int $value)
    {
        if ($value < 0 || $value > 100) {
            throw new InvalidArgumentException('Percentage must be between 0 and 100');
        }
    }
}
~~~

A PHPUnit test can verify both valid and invalid boundaries:

~~~php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

final class PercentageTest extends TestCase
{
    public function testAcceptsBoundaryValues(): void
    {
        self::assertSame(0, (new Percentage(0))->value);
        self::assertSame(100, (new Percentage(100))->value);
    }

    public function testRejectsValueAboveMaximum(): void
    {
        $this->expectException(InvalidArgumentException::class);

        new Percentage(101);
    }
}
~~~

The test says what callers may rely on. It does not inspect a private property or require a particular exception message unless that message is part of the contract.

## Arrange, Act, Assert

Keep one primary action per test. Arrange the smallest fixture, invoke the behavior once, and assert the outcome and important side effects. Several assertions can be appropriate when they describe one result, such as a parsed command containing three fields. Split a test when assertions represent independent behaviors or produce confusing failures.

Use descriptive names that include condition and result: testExpiredTokenIsRejected, testOwnerCanEditDocument, or testDuplicateKeyReturnsOriginalResult. Avoid names such as testWorks that do not explain the contract.

Prefer strict assertions. A loose comparison can let a string, integer, null, or false value pass unexpectedly. Assert an exception class and structured error code when the caller relies on them; avoid asserting incidental stack traces or full dynamic messages.

## Dependencies and Test Seams

Inject dependencies whose behavior must be controlled: a clock, random source, repository, HTTP client, filesystem, or publisher. A seam is a place where a test can substitute a controlled collaborator.

~~~php
<?php

declare(strict_types=1);

interface Clock
{
    public function now(): DateTimeImmutable;
}

final readonly class ExpiryPolicy
{
    public function __construct(private Clock $clock)
    {
    }

    public function isExpired(DateTimeImmutable $expiresAt): bool
    {
        return $expiresAt <= $this->clock->now();
    }
}
~~~

A fake clock makes boundary tests repeatable. Global time calls, environment lookups, and static singletons make tests order-sensitive and hide dependencies. Wrap them at an application boundary instead of mocking the language runtime.

## Testing Errors and Boundaries

Invalid input, missing records, authorization failures, duplicate operations, timeouts, and dependency errors are first-class behaviors. Test both the returned error and the absence of forbidden side effects. For example, a rejected payment command must not publish a captured-payment event.

Test boundaries that have caused incidents: zero and maximum amounts, empty collections, Unicode text, timezone transitions, integer overflow, duplicate keys, stale versions, and retry behavior. Property-based tests and data providers can cover families of inputs without hiding the meaning of the invariant.

## Test Data Builders

Builders reduce noisy fixture setup while keeping the meaningful variation visible. A builder should create valid defaults and allow a test to override fields relevant to its scenario. It must not silently erase a value the test intends to exercise.

Do not use real customer records. Synthetic fixtures are safer and make expected values explicit. Keep builders deterministic and avoid a single “god fixture” that configures every possible field.

## Avoiding Over-Mocking

Mocking every collaborator makes tests pass while the real collaboration is broken. Use a stub to supply a value, a fake to implement a small working substitute, and a mock or spy only when an interaction itself is the behavior under test. Prefer an in-memory repository for a domain service when it preserves the relevant repository contract; use an integration test for SQL, constraints, and serialization.

When a mock expectation describes private call order rather than a public side effect, it is coupling the test to implementation. A refactor that preserves behavior should not require rewriting dozens of interaction assertions.

## PHPUnit Organization and Speed

Use PHPUnit's test discovery, assertions, data providers, fixtures, and lifecycle hooks deliberately. Keep per-test setup small. Run the fast unit suite on every change and separate slower groups for integration, feature, or end-to-end tests.

Do not use process-global mutable state to share fixtures. If static analysis or a coding standard catches a bug, keep the test that proves the behavior and let the tool provide its different evidence. Tests and static analysis answer different questions.

## Failure and Threat Analysis

* **Hidden dependency:** global clock or configuration changes results by machine. Inject the dependency.
* **Over-mocking:** expected calls pass while the real adapter is broken. Test the adapter at its integration boundary.
* **Weak assertions:** a test passes for any truthy value. Use strict, contract-level assertions.
* **Fixture coupling:** a shared mutable fixture makes order matter. Create isolated data per test.
* **Missing side-effect assertion:** an invalid command still sends a message. Assert the forbidden interaction.
* **Slow unit suite:** accidental database or network calls enter a unit test. Keep boundaries explicit and fail fast.

## Testing Unit Tests

The test runner itself should fail on errors and unexpected output. Run unit tests in isolation and in a random or parallel order where supported. Monitor runtime and quarantine flaky tests only with a tracked owner and removal plan; hiding a failure permanently removes evidence.

Mutation testing can reveal weak assertions later in the volume. If changing a branch does not make any test fail, add a test for the missing behavior or remove the unneeded branch.

## Exercises

1. Write unit tests for a money value object covering currency, zero, negative, maximum, rounding, and equality.
2. Introduce a Clock interface into code that calls time() and test an exact expiry boundary.
3. Replace a test that mocks a repository with an in-memory fake. Identify which repository guarantees still require an integration test.
4. Add a negative assertion proving that an invalid command publishes no event.

## Review Questions

1. What makes a unit test isolated without making it meaningless?
2. Which dependencies should usually be injected for deterministic tests?
3. Why are strict assertions safer at type boundaries?
4. When is an interaction itself the behavior under test?
5. What does over-mocking hide?
6. How can mutation testing improve a unit suite?

## Summary

Unit tests give fast evidence about a small behavior. Inject time, randomness, storage, and network boundaries; use clear arrange-act-assert tests with strict assertions; cover errors and side-effect absence; and choose stubs, fakes, mocks, and integration tests according to the contract. Keep fixtures deterministic and let the suite expose design coupling.

## References

- [PHPUnit Documentation](https://docs.phpunit.de/)
- [PHPUnit Test Doubles](https://docs.phpunit.de/en/11.5/test-doubles.html)
- [PHPUnit Data Providers](https://docs.phpunit.de/en/11.5/writing-tests-for-phpunit.html#data-providers)
- [PHP Manual: DateTimeImmutable](https://www.php.net/manual/en/class.datetimeimmutable.php)

