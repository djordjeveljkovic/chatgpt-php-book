---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 174
title: Flaky Tests
slug: flaky-tests
status: complete
summary: ../../_ai/chapter-summaries/174-flaky-tests-summary.md
---

# Chapter 174 — Flaky Tests

## Why This Matters

A flaky test passes and fails without a relevant code change. It consumes investigation time, teaches developers to ignore red builds, and can conceal a real regression behind repeated retries. Flakiness is a reliability defect in the test system, even when the product code is correct.

Treat the test result as evidence with a cause. A test that fails because a remote service is unavailable is an environment failure; a test that races a queue is a test or product synchronization defect; a test that shares mutable rows with another worker has an isolation defect. Classifying the cause determines the repair.

## Sources of Nondeterminism

Common sources include:

- wall-clock time, time zones, daylight-saving transitions, and expiry boundaries;
- random identifiers or random input without a recorded seed;
- shared database rows, files, caches, queues, or browser sessions;
- parallel workers that assume a global order;
- asynchronous jobs and eventual consistency;
- network, DNS, CPU, memory, and external service variation;
- leaked process state, environment variables, static properties, or singletons;
- tests that depend on execution order or a previous test's cleanup.

A retry can show that a failure is intermittent, but it does not explain why. Capture the first failure before rerunning.

## Replace Time and Randomness

Inject time into code whose result depends on now. The test can move the clock deliberately instead of sleeping.

~~~php
<?php

declare(strict_types=1);

interface Clock
{
    public function now(): DateTimeImmutable;
}

final class FakeClock implements Clock
{
    public function __construct(private DateTimeImmutable $current)
    {
    }

    public function now(): DateTimeImmutable
    {
        return $this->current;
    }

    public function advance(DateInterval $interval): void
    {
        $this->current = $this->current->add($interval);
    }
}

function sessionIsAlive(Clock $clock, DateTimeImmutable $expiresAt): bool
{
    return $clock->now() < $expiresAt;
}
~~~

A PHPUnit test can set a fixed instant, call the operation, advance the fake clock, and assert expiry. Use a monotonic clock for elapsed-duration logic where wall-clock corrections should not change the result. Inject a random source or record a property-test seed; never make a flaky test reproducible by hard-coding one accidental random value without testing the broader domain.

## Wait for State, Not Time

A fixed sleep guesses how long an asynchronous operation will take. It may be too short on a slow CI runner and unnecessarily long when the operation finishes quickly. Poll a meaningful condition with a deadline and include the last observed state in the failure.

~~~php
<?php

declare(strict_types=1);

function waitUntil(callable $condition, int $timeoutMs, int $intervalMs = 25): void
{
    $deadline = hrtime(true) + ($timeoutMs * 1_000_000);
    $lastError = null;

    do {
        try {
            if ($condition()) {
                return;
            }
        } catch (Throwable $exception) {
            $lastError = $exception;
        }

        usleep($intervalMs * 1_000);
    } while (hrtime(true) < $deadline);

    throw new RuntimeException(
        'Condition did not become true before the deadline',
        previous: $lastError,
    );
}
~~~

The condition should be observable and bounded: a job status, a row version, or a message in a test sink. Polling a random delay or swallowing every exception can hide a permanent failure. If the test controls the queue, a deterministic drain hook is often better than polling a production worker.

## Isolation and Parallel Runs

Give every test worker a unique namespace for database schemas, cache prefixes, files, queues, and external resources. A transaction rollback is useful only when all work uses the same connection and no asynchronous task outlives the transaction. Clean up resources in a finally block and run periodic orphan cleanup.

Order-dependent tests often rely on leaked state. Run the suite in a randomized order in CI and preserve the seed when a failure appears. Run a failing test alone, then run it repeatedly and in parallel with the suspected neighbor. This narrows whether the defect is local, order-sensitive, or a resource race.

Browser tests need independent contexts when cookies, local storage, or service workers can affect behavior. API tests need unique idempotency keys and fixture identifiers. Filesystem tests need temporary directories rather than a shared path such as the repository root.

## Diagnose Before Retrying

Capture the test name, worker, seed, build and environment identifiers, timestamps, relevant request IDs, queue state, database transaction state, and resource ownership. Keep screenshots, logs, and traces bounded and redacted. A test retry should preserve the first failure and mark the result as a retry, not replace the original evidence.

Useful experiments include:

1. Run the test repeatedly with a fixed order and fixed seed.
2. Run it alone and with parallel workers.
3. Replace external services with a deterministic test server.
4. Freeze time and assign unique resource names.
5. Increase diagnostics without increasing the timeout.
6. Compare failures across PHP, database, browser, and operating-system versions.

A failure that disappears after increasing a timeout is evidence of a race or capacity problem, not proof that the timeout was correct.

## Quarantine and Ownership

Quarantine can keep a known flaky test from blocking unrelated delivery, but it must be temporary. Record the owner, issue, suspected cause, first observed build, affected environments, and removal deadline. Run the quarantined test separately and report its result prominently. Do not silently skip it or allow a permanently red test lane.

Track flake rate, first-attempt pass rate, retry pass rate, mean time to repair, and top affected suites. A test that fails only under a particular database or browser version may reveal a product compatibility defect. Metrics make the cost visible and help prioritize repairs.

## CI and Production Signals

Use CI retries sparingly and distinguish infrastructure retry from test retry. A worker may be lost before a test result exists; that is different from a test assertion failure. Pin or record tool versions, provide enough CPU and memory, and verify dependency health before executing tests. Avoid making the entire suite depend on one overloaded shared service.

When a production incident reveals an untested race, add a deterministic regression test at the lowest layer that can observe it. An end-to-end test may prove the symptom, while a unit or integration test can prove the synchronization rule more reliably.

## Common Mistakes

- Adding sleeps or retries without identifying the race.
- Ignoring the first failure after a retry passes.
- Sharing mutable fixtures across parallel workers.
- Treating random data and wall-clock time as harmless globals.
- Quarantining a test with no owner or removal date.
- Increasing timeouts until a broken synchronization appears healthy.
- Deleting a flaky test instead of preserving the behavior it was meant to protect.

## Senior Engineer Thinking

Flakiness is a measurement problem and an engineering signal. Make time, randomness, state, and external dependencies controllable; isolate resources; preserve first-failure evidence; and repair the cause. A trustworthy suite fails for a reason that the team can act on.

## Exercises

1. Refactor an expiry test that sleeps into one using an injected fake clock.
2. Design a parallel test namespace for a database, Redis prefix, filesystem directory, and queue.
3. Build a failure report containing a random seed, worker ID, correlation ID, and last observed asynchronous state.
4. Define a quarantine policy with ownership and a removal deadline.

## Review Questions

1. Why is a passing retry not a diagnosis?
2. Which kinds of time should use a wall clock and which should use a monotonic clock?
3. When does transaction rollback fail to isolate a test?
4. What evidence should be captured before retrying a failure?
5. How can flake metrics guide engineering work?

## Summary

Flaky tests fail without a relevant code change and damage trust in delivery. Classify the cause, inject time and randomness, wait for observable state, isolate parallel resources, preserve first-failure diagnostics, and use quarantine only with ownership and a deadline. Repair the synchronization or environment defect rather than masking it with sleeps and retries.

## References

- [PHPUnit documentation](https://docs.phpunit.de/)
- [PHP hrtime()](https://www.php.net/manual/en/function.hrtime.php)
- [PHP DateTimeImmutable](https://www.php.net/manual/en/class.datetimeimmutable.php)
- [Martin Fowler: Eradicating Non-Determinism in Tests](https://martinfowler.com/articles/nonDeterminism.html)
- [Google Testing Blog: Flaky Tests](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)

