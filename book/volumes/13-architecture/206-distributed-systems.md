---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 206
title: Distributed Systems
slug: distributed-systems
status: complete
summary: ../../_ai/chapter-summaries/206-distributed-systems-summary.md
---

# Chapter 206 — Distributed Systems

A distributed system has independent processes that communicate over a network and can fail separately. The network can delay, duplicate, reorder, or drop messages. A process can be healthy while its dependency is unavailable, and a response can be lost after the remote side commits the operation.

The correct design starts with the failure model and business invariant. Adding services or retries without that model creates duplicate charges, stale reads, retry storms, and workflows that cannot be repaired.

## Why this matters

A PHP request that calls three services has more than three function calls. It has timeouts, authentication, DNS, connection pools, partial success, deploy skew, and uncertain outcomes. A user may retry after a timeout even though the remote service completed the work.

State the contract before choosing a protocol:

- What is the operation's identity and idempotency key?
- Which side owns the truth and which data is a projection?
- What consistency does the caller need?
- Which failures are retryable and for how long?
- What is the timeout budget and user-visible outcome?
- How can operators reconcile an ambiguous result?

## Timeouts and bounded retries

Every network call needs a connect timeout, total deadline, and response-size limit. A retry must have a budget, backoff with jitter, and a classification of failures. Do not retry validation errors or non-idempotent operations without an idempotency contract.

```php
<?php

declare(strict_types=1);

final readonly class RetryPolicy
{
    public function __construct(
        public int $maxAttempts,
        public int $baseDelayMilliseconds,
    ) {
        if ($maxAttempts < 1 || $baseDelayMilliseconds < 1) {
            throw new InvalidArgumentException('Invalid retry policy');
        }
    }

    public function delay(int $attempt): int
    {
        $exponential = $this->baseDelayMilliseconds * (2 ** max(0, $attempt - 1));
        $jitter = random_int(0, $this->baseDelayMilliseconds);

        return min($exponential + $jitter, 30_000);
    }
}

function runWithRetry(callable $operation, RetryPolicy $policy): mixed
{
    $last = null;
    for ($attempt = 1; $attempt <= $policy->maxAttempts; $attempt++) {
        try {
            return $operation();
        } catch (RetryableTransportFailure $exception) {
            $last = $exception;
            if ($attempt === $policy->maxAttempts) {
                break;
            }
            usleep($policy->delay($attempt) * 1_000);
        }
    }

    throw $last ?? new RuntimeException('Operation failed');
}
```

This is a policy example, not a recommendation to sleep inside a web worker without a deadline. Queue retries are often safer for slow recovery. A retry can amplify an outage; use circuit breaking, load shedding, and a bounded caller budget when appropriate.

## Ambiguous outcomes and idempotency

A timeout does not tell the caller whether the remote operation committed. Send a stable idempotency key, persist the request state, and query or reconcile by that key. A response such as “accepted” must not be presented as “completed” unless the owner guarantees that state.

For a payment, store the business operation and provider key before calling the provider. On retry, ask for the existing provider result rather than creating a new charge. For a notification, duplicate delivery may be acceptable if the message is idempotent or deduplicated.

## Consistency and ownership

Choose one owner for each piece of mutable truth. Other services keep projections and accept propagation delay. Do not update the same business fact independently in two databases and hope periodic jobs will resolve conflicts.

Strong consistency has latency and availability costs; eventual consistency has stale-read and reconciliation costs. Make freshness visible in APIs and UI. A read-after-write requirement may be satisfied by routing to the owner, waiting for a projection version, or returning a pending state.

Transactions do not span arbitrary services. Use a saga or workflow with explicit steps and compensating actions when a process spans owners. A compensation is a new business action, not a magical rollback; it can fail and may require manual recovery.

## Failure isolation

Bulkheads limit how many workers, connections, or queue slots one dependency can consume. Circuit breakers stop calls after repeated failures and allow controlled probes later. Rate limits protect both the caller and the dependency. Backpressure prevents an outage from creating an unbounded backlog.

Keep fallback behavior safe. Returning stale catalog data may be acceptable; granting access, charging twice, or silently dropping an audit event is not. Fail closed for authorization and financial invariants, and define degraded behavior for non-critical features.

## Messages, clocks, and observability

Use correlation IDs and causation IDs to follow one business operation across HTTP, queues, and workers. Include event IDs, attempt numbers, owner, and outcome in structured telemetry. Redact tokens and personal data.

Do not compare wall-clock timestamps as if they prove ordering across hosts. Use monotonic durations locally and sequence numbers, versions, or broker offsets for ordering. Clock skew affects expiry and signatures; define tolerance and monitor it.

## Testing and operations

Test timeout, duplicate, delayed, reordered, malformed, unauthorized, and partial-success scenarios. Use fault injection or a controlled proxy to simulate failures. Test retry budgets and ensure one outage does not create a retry storm. Run reconciliation jobs in a safe environment and make their actions auditable.

Monitor dependency latency and error rate, circuit state, queue age, saturation, idempotency conflicts, reconciliation volume, and distributed trace completeness. A health check should distinguish process liveness from dependency readiness; see [Chapter 263 — Health Checks](../17-production-engineering/263-health-checks.md).

## Exercises

1. Design a payment operation for an ambiguous timeout. Define the idempotency key, stored states, retry policy, and reconciliation path.
2. Draw a saga for order, inventory, payment, and shipment. Identify compensating actions and irrecoverable steps.
3. Set a 2-second user deadline across three downstream calls. Allocate budgets and decide which work moves to a queue.

## Review questions

- Why is a network call different from a local function call?
- When is a retry unsafe or harmful?
- How does idempotency resolve an ambiguous outcome?
- What does a service projection trade for independent ownership?
- Why are compensation and rollback different?
- Which signals reveal a retry storm or dependency saturation?

## Summary

Design distributed systems from explicit failure and ownership models. Bound calls, retries, queues, and resources; make operations idempotent; choose consistency deliberately; isolate failures; preserve traceable evidence; and provide reconciliation for partial and ambiguous outcomes.

## References

- [Martin Kleppmann: Designing Data-Intensive Applications](https://dataintensive.net/)
- [Google SRE: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Martin Fowler: Idempotent Receiver](https://martinfowler.com/articles/patterns-of-distributed-systems/idempotent-receiver.html)
- [PHP manual: random_int](https://www.php.net/manual/en/function.random-int.php)
