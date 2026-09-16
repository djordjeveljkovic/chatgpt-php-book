---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 239
title: Retries
slug: retries
status: complete
summary: ../../_ai/chapter-summaries/239-retries-summary.md
---

# Chapter 239 — Retries

## Why This Matters

A retry is a new attempt after an earlier attempt failed or became uncertain. It can recover from a transient connection reset or overloaded dependency, but it also repeats work, consumes time, and adds load to a system that may already be unhealthy.

Retry only when the caller has a reason to expect that another attempt can succeed, the remaining deadline permits it, and repeating the operation is safe or has an idempotency mechanism. “The request failed” is not enough information to justify a retry.

## Classify the Outcome

A client should distinguish at least:

* **success:** the desired result is known;
* **permanent failure:** invalid input, missing authorization, or an unsupported operation;
* **transient failure:** a bounded overload response, connection failure, or temporary unavailability;
* **unknown completion:** the receiver may have completed a side effect before the response was lost;
* **deadline exhausted:** no time remains for a useful attempt.

The transport may expose status codes or exception types, but the application maps them to a policy. A 500 response can be a transient server error or a deterministic bug. A timeout can occur before the server receives a request or after it commits it. Keep “retryable” and “safe to repeat” as separate decisions.

## Safe and Unsafe Operations

Read-only operations are often easier to retry, provided the read itself is acceptable to repeat and the result can be stale. A database `SELECT` is not automatically cheap or harmless under overload.

An effecting operation needs one of:

* a protocol-defined idempotent operation;
* a client-supplied idempotency key recorded by the owner;
* a compare-and-set or unique constraint that makes repeats converge;
* a query-and-reconcile path for unknown completion;
* a domain rule that explicitly permits duplicate effects.

Do not infer safety from an HTTP verb alone. A `POST` may be idempotent by contract; a `GET` that triggers a purchase is still an unsafe effect.

## Retry Budgets

Retries must fit the parent deadline and an attempt budget. If the first attempt used 500 ms of an 800 ms request budget, a second attempt needs time for its own work and response; it cannot receive a fresh 800 ms timeout.

Use a limit on attempts and, where useful, a limit on total retry work per caller, tenant, dependency, and time window. A retry budget prevents a high-volume outage from multiplying traffic without bound. Budget accounting belongs near the policy boundary so nested libraries do not each invent independent retries.

```text
original request
  ├─ attempt 1: 250 ms
  ├─ wait:       80 ms
  └─ attempt 2: remaining deadline
```

The wait is part of the request's latency and must be included in the deadline. See [Chapter 238 — Timeouts](./238-timeouts.md) and [Chapter 240 — Backoff](./240-backoff.md).

## A Small Retry Policy

Make the policy data visible and inject the sleeping mechanism so tests do not need real time:

~~~php
<?php

declare(strict_types=1);

enum AttemptOutcome: string
{
    case Success = 'success';
    case TransientFailure = 'transient_failure';
    case PermanentFailure = 'permanent_failure';
    case UnknownCompletion = 'unknown_completion';
}

interface Sleeper
{
    public function sleepMilliseconds(int $milliseconds): void;
}

/** @param callable(int): AttemptOutcome $attempt */
function retry(
    callable $attempt,
    int $maxAttempts,
    int $maxElapsedMs,
    Sleeper $sleeper,
    callable $delayForAttempt,
): AttemptOutcome {
    $started = hrtime(true);

    for ($number = 1; $number <= $maxAttempts; $number++) {
        $outcome = $attempt($number);

        if ($outcome !== AttemptOutcome::TransientFailure) {
            return $outcome;
        }

        $elapsedMs = (hrtime(true) - $started) / 1_000_000;
        $delayMs = (int) $delayForAttempt($number);
        if ($number === $maxAttempts || $elapsedMs + $delayMs >= $maxElapsedMs) {
            break;
        }

        $sleeper->sleepMilliseconds($delayMs);
    }

    return AttemptOutcome::TransientFailure;
}
~~~

The example deliberately does not retry `UnknownCompletion`. The caller must reconcile that outcome using an operation identity before deciding whether another request is safe. A production policy should reject invalid limits, check remaining deadline at the transport boundary, and record attempts without logging sensitive payloads.

## Layering Retries

One operation can accidentally be retried at several layers:

```text
HTTP client: 3 attempts
application service: 3 attempts
queue transport: redeliver 5 times
provider: internal retries
```

The product may see many more attempts than any one layer expects. Choose an owner for each retry decision. Usually the layer with enough semantic context decides whether a side effect is safe, while lower layers expose failures without silently repeating business operations.

If layers must retry independently, calculate the worst case and propagate an attempt or retry budget. Record the causal chain so an incident can distinguish one request from its multiplied attempts.

## Retries and Transactions

Do not hold a database transaction open while sleeping between remote attempts. Long transactions retain locks and connections and make a retry storm more expensive. Commit local state that records an operation, perform the remote interaction under its contract, then reconcile the result in a bounded workflow.

For a local transient database serialization failure, retrying the whole transaction can be appropriate when the transaction is side-effect-free outside that database and the engine's error is explicitly classified. Recreate transaction-local state for each attempt; do not reuse a mutated object graph or stale snapshot.

## Retry-After and Server Signals

A server may communicate a retry time or rate-limit reset. Respect a valid bounded `Retry-After` or equivalent signal, subject to the caller's deadline and policy. Do not sleep for an attacker-controlled unbounded duration. If a response says the operation is permanently rejected, retrying ignores the protocol and increases load.

The server cannot know whether a client received its response. For effecting operations, return or store a durable operation identity that can be queried later.

## Testing

Test a sequence of outcomes: transient then success, repeated transient failures, permanent failure, unknown completion, deadline exhaustion, malformed retry metadata, and a sleeping function that records delays. Assert the number of attempts, elapsed-budget checks, final classification, and absence of retries for unsafe unknown outcomes.

Use a fake clock or injected elapsed-time source for boundary tests. Run one integration test that verifies the actual transport's exceptions and status mapping. Test duplicate effects at the owning service, not only the caller's retry loop.

## Security and Operations

Retries can amplify denial of service and can multiply charges, emails, reservations, or webhook deliveries. Apply per-operation and per-tenant budgets, cap concurrency, and monitor attempts per successful operation. Redact credentials and private payloads from retry logs.

Useful signals include retry rate, attempts per request, final outcome, time spent waiting, provider response category, and downstream saturation. An increase in retry success can still be unhealthy if it consumes capacity and pushes latency beyond the contract.

## Common Mistakes

* Retrying every exception or every 5xx response indiscriminately.
* Repeating an effect without an idempotency key or reconciliation path.
* Giving each retry a new full timeout.
* Sleeping inside a database transaction.
* Retrying at transport, service, queue, and provider layers without a budget.
* Ignoring server retry guidance or trusting an unbounded delay.
* Treating an unknown completion as a definite failure.
* Measuring only final success and hiding the number of attempts.

## Senior Engineer Thinking

The question is not “can this failure go away on the next attempt?” It is “what evidence says another attempt is useful, what work might already have happened, and which budget pays for it?” A retry is part of the operation's semantics and capacity model, not a generic exception handler.

## Exercises

1. Classify ten failure outcomes from an HTTP and database client as permanent, transient, unknown, or deadline-exhausted. Explain each decision.
2. Add an idempotency record to a payment request and define the result of a timeout after provider completion.
3. Trace the worst-case attempt count when an HTTP client, application service, and queue each retry. Redesign the budgets.
4. Test a transaction retry that reconstructs all transaction-local state after a serialization failure.

## Review Questions

* Why are retryability and repeat safety separate decisions?
* What does an unknown completion require before another effecting attempt?
* Why should retry waits count against the parent deadline?
* What happens when multiple layers retry independently?
* Why should a remote retry not sleep inside a database transaction?
* Which operational metric reveals retry amplification?

## Summary

Retries recover only from failures classified as transient and only within a shared deadline and bounded attempt budget. Separate permanent failure, transient failure, unknown completion, and safe repetition. Make effecting operations idempotent or reconcilable, avoid holding scarce resources while waiting, coordinate retry layers, and measure attempts as well as final outcomes.

## References

- [AWS Builders' Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Google SRE: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [RFC 9110: Retry semantics](https://www.rfc-editor.org/rfc/rfc9110#section-9.2.2)
- [Chapter 141 — Idempotency](../../volumes/09-http-and-application-development/141-idempotency.md)
