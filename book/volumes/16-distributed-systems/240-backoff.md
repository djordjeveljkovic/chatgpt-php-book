---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 240
title: Backoff
slug: backoff
status: complete
summary: ../../_ai/chapter-summaries/240-backoff-summary.md
---

# Chapter 240 — Backoff

## Why This Matters

Backoff spaces repeated attempts apart. It gives a recovering dependency time to serve existing work and prevents a group of callers from retrying in lockstep. Backoff is useful only when paired with a retry classification, a deadline, and a safe duplicate policy. A delayed unsafe duplicate is still unsafe.

Immediate retries are attractive because they appear to reduce latency. During an outage they turn every failure into another request, competing with recovery traffic and increasing queueing. A bounded, jittered schedule makes the load less synchronized and more predictable.

## Exponential Backoff

A common schedule grows with the attempt number:

```text
delay(n) = min(cap, base × 2^(n - 1))
```

For a 100 ms base and 2 second cap, the uncapped delays are 100, 200, 400, 800, 1,600, and then 2,000 ms. The cap prevents an individual delay from exceeding a useful bound, but the sum of delays still has to fit the operation's deadline.

Exponential growth is a policy, not a physical law. Choose base, multiplier, cap, and attempt count from the dependency's recovery behavior and user contract. A payment request with a one-second interactive budget needs a different schedule from a durable overnight export.

## Jitter

Without jitter, callers that fail together retry together:

```text
failure ── 1s ── retry ── 2s ── retry
failure ── 1s ── retry ── 2s ── retry
failure ── 1s ── retry ── 2s ── retry
```

Randomness spreads the attempts:

```text
failure ─ 0.7s ─ retry
failure ─ 1.1s ─ retry
failure ─ 0.9s ─ retry
```

Common policies include:

* **full jitter:** choose uniformly from zero through the exponential limit;
* **equal jitter:** preserve part of the limit and randomize the remainder;
* **decorrelated jitter:** choose the next delay from a range influenced by the previous delay.

The exact distribution matters less than making the policy bounded, measurable, and appropriate for the workload. Use a cryptographically secure random generator only when unpredictability is a security requirement; ordinary scheduling jitter can use a non-security random source or an injected deterministic generator for tests.

## A Bounded Delay Function

Keep the pure calculation separate from sleeping:

~~~php
<?php

declare(strict_types=1);

function fullJitterDelayMs(
    int $attempt,
    int $baseMs,
    int $capMs,
    callable $randomInt,
): int {
    if ($attempt < 1 || $baseMs < 0 || $capMs < $baseMs) {
        throw new InvalidArgumentException('Invalid backoff policy');
    }

    $limit = $baseMs;
    for ($step = 1; $step < $attempt && $limit < $capMs; $step++) {
        $limit = min($capMs, $limit * 2);
    }

    return $randomInt(0, $limit);
}

$delay = fullJitterDelayMs(
    attempt: 3,
    baseMs: 100,
    capMs: 2_000,
    randomInt: static fn (int $min, int $max): int => random_int($min, $max),
);
~~~

The injected callable makes the schedule testable without waiting or relying on a particular random draw. The loop avoids computing a huge power directly, although multiplication overflow must still be considered for very large caps. Validate configuration at startup rather than discovering it during an outage.

## Respecting Server Signals

An HTTP response may provide `Retry-After` or an API-specific reset time. Prefer a valid, bounded server signal when the protocol says it applies. Clamp it to the caller's remaining deadline and a local maximum. A server or intermediary cannot be allowed to make a request sleep forever.

If the response indicates a quota reset at a known time, a queue scheduler can delay work until then. Interactive callers should usually return a bounded response rather than hold a PHP-FPM worker for the entire reset interval.

## Backoff and Queues

For a queue, delaying a message is different from sleeping in the worker. Release or reschedule the message so the worker can process other work. Preserve attempt count, first-seen time, operation identity, and the reason for delay. A visibility timeout that expires while the worker sleeps can cause a second consumer to process the same message.

Do not apply the same delay to every queue class. A poison message should move to a failure transport rather than occupy a retry schedule. A high-priority recovery message may need a separate queue and worker budget from bulk work.

## Backoff and Capacity

Backoff reduces immediate retry pressure but increases completion latency and retained work. Count delayed items in queue age and service-level calculations. A system that reports an empty active queue while thousands of messages sit in delayed storage is not healthy merely because workers are idle.

When many clients back off, recovery can create a second wave as their schedules expire. Use jitter, bounded concurrency, circuit breakers, and admission control. After recovery, ramp up rather than releasing an unlimited backlog at once.

## Backoff for Locks and Transactions

Short randomized waits can help when retrying a database deadlock or serialization failure, but the transaction must be recreated and the total budget bounded. Do not sleep while holding a database lock or connection if the operation can release it first. A lock retry policy belongs with the database error classification described in [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md).

Backoff does not solve a permanent conflict. If two commands continually violate an invariant, route them through an ordering or serialization policy rather than retrying indefinitely.

## Testing the Schedule

Test the pure delay function with a deterministic random callable. Assert:

* delays are nonnegative and no greater than the cap;
* the exponential limit reaches the cap without overflow;
* attempt one uses the configured base limit;
* random bounds are inclusive or exclusive according to the chosen contract;
* total delay plus work fits the parent deadline;
* server-provided delays are clamped;
* queue attempt metadata survives rescheduling.

For an integration test, use a fake dependency that fails a controlled number of times and record request timestamps. Verify that the client does not sleep while holding a transaction or worker resource that should have been released.

## Security and Operations

Random backoff is not an access-control mechanism. Do not use it to conceal authorization failures or to make brute-force protection the only defense. Rate limits, quotas, and account lock policies need explicit security semantics.

Monitor delay distributions, retry attempts, delayed queue age, final outcomes, and dependency load. An unusually high delay can indicate recovery or a misconfigured cap; a low delay with high retry volume can indicate synchronization or a bypassed policy.

## Common Mistakes

* Retrying immediately after every failure.
* Using identical deterministic delays for all callers.
* Letting a server-controlled delay exceed the request deadline.
* Sleeping inside a worker while the queue lease or database lock remains held.
* Applying a transient retry schedule to permanent validation failures.
* Computing exponential powers without bounds or overflow checks.
* Ignoring delayed queue age when reporting backlog.
* Releasing an entire recovered backlog at once.

## Senior Engineer Thinking

Backoff is a coordination policy. Ask which resource needs time, whether the caller should wait at all, how many peers will retry, and how delayed work is observed. A good schedule reduces pressure without making the business contract or recovery state invisible.

## Exercises

1. Compare no jitter, full jitter, and equal jitter for 1,000 callers. Plot the retry distribution and maximum simultaneous attempts.
2. Choose a backoff schedule for an interactive API and a bulk queue. Include deadline, cap, attempt count, and delayed-work visibility.
3. Test a deadlock retry that releases all database resources before waiting and reconstructs the transaction afterward.
4. Design a recovery ramp for a queue with 100,000 delayed messages after an upstream outage.

## Review Questions

* Why is backoff insufficient without retry classification and idempotency?
* What problem does jitter solve?
* Why should queue workers reschedule instead of sleeping for long delays?
* How can delayed work make backlog metrics misleading?
* Which limits prevent a backoff schedule from exceeding its contract?
* Why is a poison message not a good candidate for repeated exponential retries?

## Summary

Backoff spaces retry attempts so dependencies can recover and callers do not synchronize into a retry storm. Use a bounded exponential policy with jitter, respect safe server signals, include delay in the parent deadline, release scarce resources before waiting, and make delayed queue work visible. Test the pure schedule deterministically and monitor both attempts and completion latency.

## References

- [AWS Builders' Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Google SRE: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [PHP manual: `random_int`](https://www.php.net/manual/en/function.random-int.php)
- [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md)
