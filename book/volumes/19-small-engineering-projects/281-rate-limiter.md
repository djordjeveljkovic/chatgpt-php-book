---
book: The Complete Modern PHP Engineering Book
volume: 19
volume_title: SMALL ENGINEERING PROJECTS
chapter: 281
title: Rate Limiter
slug: rate-limiter
status: complete
summary: ../../_ai/chapter-summaries/281-rate-limiter-summary.md
---

# Chapter 281 — Rate Limiter

## Why This Matters

A rate limiter decides whether work may enter a system. That decision protects capacity, prevents accidental retry storms, and can enforce a product or security policy. It also sits on a trust boundary: a wrong key can let one client exhaust another client’s budget, while an unsafe fail-open choice can turn a dependency outage into an application outage.

“Allow ten requests per minute” is not a complete design. We need to define ten what, per whom, for which operation, measured by which clock, stored where, and with what behavior when the limiter’s storage is unavailable. This small project makes admission control concrete.

## Define the Contract

Begin with a policy table:

| Operation | Identity | Limit | Cost | Failure mode |
| --- | --- | --- | --- | --- |
| public search | authenticated account, otherwise trusted client class | 60/minute, burst 20 | 1 token | fail closed for abuse control |
| password reset | account and network risk key | 5/hour | 1 token | fail closed |
| internal report | service identity | 10/minute, burst 2 | 1 token | return temporary failure |
| health probe | authorized monitor | separate small budget | 1 token | isolated limiter path |

The identity is part of the security contract. Do not use an arbitrary `X-Forwarded-For` value as an account identity. The edge must define which proxy addresses are trusted and how the original client address is normalized. Prefer authenticated account, tenant, API key, or service identity when the policy is about a principal; use network identity only when that is the intended fallback.

## Choose the Limiting Model

Three common models have different semantics:

* fixed window counts requests in discrete periods. It is simple but permits a burst at a boundary;
* sliding window counts recent events more smoothly but stores more history or an approximation;
* token bucket accumulates permission up to a burst capacity and spends tokens per request. It expresses both sustained rate and bounded burst.

For public search, use a token bucket. Let `capacity` be the maximum token count, `refillRate` the tokens per second, and `cost` the request cost. At time `now`:

~~~text
elapsed = max(0, now - lastRefill)
refilled = min(capacity, tokens + elapsed × refillRate)
if refilled >= cost:
    tokens = refilled - cost
    allow
else:
    tokens = refilled
    reject and calculate retry delay
lastRefill = now
~~~

The bucket is not a concurrency lock. It is an admission policy. The update of `tokens` and `lastRefill` must be atomic for one key, or two simultaneous requests can both spend the same token.

## Use a Precise Representation

Do not use floating-point tokens when a request at a boundary must be deterministic. Store microtokens and integer microseconds, or use a bounded rational representation. The example is an in-process policy object; a shared implementation must move the same transition into an atomic store operation.

~~~php
<?php

declare(strict_types=1);

final readonly class BucketState
{
    public function __construct(
        public int $microtokens,
        public int $lastRefillMicros,
    ) {
        if ($microtokens < 0 || $lastRefillMicros < 0) {
            throw new InvalidArgumentException('Bucket state is invalid');
        }
    }
}

final readonly class LimitDecision
{
    public function __construct(
        public bool $allowed,
        public BucketState $state,
        public int $retryAfterSeconds,
    ) {
    }
}

function consumeToken(
    BucketState $state,
    int $nowMicros,
    int $capacityMicrotokens,
    int $refillMicrotokensPerSecond,
    int $costMicrotokens = 1_000_000,
): LimitDecision {
    if ($nowMicros < 0 || $capacityMicrotokens <= 0 || $refillMicrotokensPerSecond <= 0 || $costMicrotokens <= 0) {
        throw new InvalidArgumentException('Bucket parameters are invalid');
    }

    $refillAtMicros = max($state->lastRefillMicros, $nowMicros);
    $elapsedMicros = $refillAtMicros - $state->lastRefillMicros;
    $refilled = min(
        $capacityMicrotokens,
        $state->microtokens + intdiv($elapsedMicros * $refillMicrotokensPerSecond, 1_000_000),
    );

    if ($refilled >= $costMicrotokens) {
        return new LimitDecision(
            true,
            new BucketState($refilled - $costMicrotokens, $refillAtMicros),
            0,
        );
    }

    $missing = $costMicrotokens - $refilled;
    $delayMicros = intdiv($missing * 1_000_000 + $refillMicrotokensPerSecond - 1, $refillMicrotokensPerSecond);

    return new LimitDecision(
        false,
        new BucketState($refilled, $refillAtMicros),
        max(1, (int) ceil($delayMicros / 1_000_000)),
    );
}
~~~

This example assumes arithmetic fits the chosen integer type and that the caller validates the capacity, rate, and cost relationship. Production code should bound elapsed time, guard multiplication overflow, and define what happens after a long sleep or a clock correction. A rejected request still advances the refill timestamp and stores the refilled balance; otherwise repeated denials could repeatedly reapply the same elapsed interval.

## Select the Clock Carefully

Within one PHP process, a monotonic clock is useful for measuring elapsed time. A bucket shared by FPM workers or hosts needs a timestamp that the shared store can interpret consistently. Options include using the store’s server time, using coordinated wall-clock time with bounded skew, or using a lease/service that owns time progression.

Do not persist a process-local monotonic counter and compare it in another process. Do not let a backward wall-clock jump create tokens. Clamp negative elapsed time to zero, record clock anomalies, and choose a policy for large forward jumps. A large jump may legitimately refill a bucket, but it must not bypass a maximum capacity.

## Make the Update Atomic

An FPM application cannot safely implement a global limit with a PHP array. Each worker has its own memory, and the state disappears when the worker exits. A shared store such as Redis or a database can hold the state, but a read-modify-write sequence is still race-prone unless it is transactional or executed as one atomic server-side operation.

The shared operation should:

1. load the bucket for one canonical key;
2. obtain a trusted current time or receive a validated time value;
3. refill and clamp the balance;
4. spend the cost or leave the balance unchanged on rejection;
5. write the new state with an expiry longer than the refill horizon;
6. return allow/deny, remaining capacity, and retry information.

Use a bounded key format and a retention policy. A key that never expires is a memory leak; an expiry shorter than the time needed to refill changes policy after idle periods. Namespace keys by environment and policy version so a limit change does not reinterpret old state accidentally.

## Scope and Fairness

A single global bucket is easy to implement and easy to abuse. Scope limits deliberately:

~~~text
tenant → account → API credential → operation → risk class
                         ↓
                  canonical limiter key
~~~

Use separate budgets for expensive and cheap operations. A search request should not consume the same budget as a password-reset attempt. For multi-tenant systems, enforce tenant and account limits independently when both fairness and abuse isolation matter. A user-wide limit does not replace an endpoint-specific security limit.

Rate limiting can also create unfairness. One popular tenant may consume a shared connection or worker pool even while its own requests are within quota. Pair admission control with the resource bulkheads and backpressure described in Chapters 246–248. Limiting request count does not directly limit request cost, response size, database time, or concurrent in-flight work; charge according to the resource being protected when necessary.

## HTTP and Error Semantics

An HTTP limiter should return a stable status and machine-readable reason. `429 Too Many Requests` is appropriate for a policy denial; a limiter storage outage may require a different temporary failure or a fail-closed 429 depending on the protected capability. Include a bounded `Retry-After` value when a retry can be meaningful, and document whether it is seconds or an HTTP date.

Do not instruct every client to retry immediately. The client needs a backoff policy, and the server needs protection from synchronized retries. For an authenticated API, return remaining budget only if that information does not disclose another principal’s state. Keep response timing and error detail from becoming an account-enumeration channel for sensitive operations.

## Fail-Open or Fail-Closed

There is no universal answer. Decide per operation:

| Capability | Storage unavailable | Reason |
| --- | --- | --- |
| password reset | fail closed | abuse prevention is the safety property |
| authenticated read | bounded degraded path or fail closed | balance availability and capacity |
| payment mutation | fail closed | uncontrolled retries can create harmful load or effects |
| health probe | use an isolated path | the limiter must not hide process health |

Bound a degraded path. “Fail open temporarily” without a duration, local emergency budget, alert, and owner is an outage amplifier. Conversely, putting every request behind an unavailable limiter can create a total outage. Record the decision and test it as an explicit failure mode.

## Tests That Prove the Policy

Use a fake clock and deterministic state for unit tests:

* an initially full bucket accepts up to capacity;
* a partial refill accepts only the earned integer amount;
* adjacent boundary times produce the documented result;
* a backward clock does not create tokens;
* a long idle period stops at capacity;
* a request cost greater than capacity is rejected or forbidden by configuration;
* rejected requests do not spend tokens;
* two concurrent updates for one key cannot both spend the same token;
* different canonical keys do not share state;
* key expiry and policy-version changes follow the retention contract;
* storage timeout follows the operation’s fail-open/closed policy;
* response status and retry metadata remain stable.

Run integration tests against the actual shared-store primitive. A unit-tested token calculation cannot prove a Redis script, database lock, network timeout, or serialization behavior. Load tests should include hot keys, many cold keys, burst traffic, storage latency, and retrying clients.

## Observe Without Leaking Identity

Measure allowed, denied, storage-error, and degraded decisions by bounded policy, operation, and outcome dimensions. Track limiter latency, hot-key contention, state-store capacity, key count, expiry behavior, and rejected work. Do not use raw user IDs or arbitrary IP addresses as metric labels. Logs may include a redacted or hashed correlation value only when the retention and privacy policy allows it.

Alert on storage errors, fail-open decisions, unexpected policy-version misses, and sustained denial changes. A high denial rate may be normal during a deliberate attack or a popular event; pair it with capacity, error, and identity-class signals before paging.

## Rollout and Change Safety

Start in observe-only mode, but ensure observation does not mutate the bucket or accidentally call the protected operation twice. Compare predicted decisions with current traffic, sample safely, and bound the cardinality of keys in diagnostics. Then enable one low-risk operation with a conservative budget and a tested emergency switch.

Changing a limit is a policy deployment. Version the configuration, record the owner and approval, and decide whether existing buckets are migrated, namespaced, or allowed to expire. During mixed-version rollout, all workers must agree on the key format and state representation. A new worker that interprets microtokens as whole tokens can bypass the policy even when the code is individually correct.

## Common Mistakes

* Counting arbitrary client-supplied IP headers as trusted identity.
* Using a PHP worker-local array for a cross-worker limit.
* Treating a read-then-write sequence as atomic because it is inside one request.
* Using floating point at a boundary where integer policy is required.
* Persisting process-local monotonic time across hosts.
* Allowing a backward clock correction or large arithmetic overflow to mint tokens.
* Sharing one budget across operations with very different cost or security impact.
* Returning `429` without a retry contract or causing synchronized retries.
* Treating storage failure as universally fail-open or universally fail-closed.
* Measuring raw user IDs or IPs as unbounded metric labels.
* Enabling a new policy while old workers use a different key or unit representation.

## Senior Engineer Thinking

The senior question is not “how many requests per minute should we allow?” It is “which resource and principal are we protecting, what burst and fairness semantics are promised, how is state updated atomically, what does time mean across workers, and what happens when the limiter itself is unavailable?”

A limiter is a control loop at the system boundary. It can reduce overload only when its key, clock, storage, cost model, and failure policy match the resource being protected. Keep the decision deterministic, state bounded, behavior observable, and degraded operation explicit.

## Exercises

1. Implement fixed-window, sliding-window, and token-bucket policies over the same request trace. Compare boundary bursts and state size.
2. Design canonical keys for a multi-tenant API with account, API key, operation, and risk-class limits. State which identity sources are trusted.
3. Write the atomic shared-store transition for one bucket and list timeout, retry, expiry, and clock-anomaly behavior.
4. Choose fail-open or fail-closed behavior for five capabilities, including a login, a report, a health probe, and a payment mutation.
5. Load-test a hot key and a large cold-key population. Define capacity, latency, storage-error, and fairness signals.

## Review Questions

* Why is a rate limit an admission-control contract rather than a counter?
* How do fixed windows, sliding windows, and token buckets differ?
* Why must a shared bucket update be atomic?
* Which clocks are appropriate within a process and across processes?
* What makes a limiter key canonical and trustworthy?
* Why should cost sometimes represent resource work rather than request count?
* How should a service decide between fail-open and fail-closed behavior?
* Why are unit tests insufficient for a distributed limiter store?
* Which metrics reveal hot keys without exposing raw identities?
* What must be versioned when a limiter policy changes during mixed deployment?

## Summary

A rate limiter is a precise admission-control policy. Define the protected resource, principal, rate, burst, cost, clock, storage, and failure behavior. Choose a model whose semantics fit the workload, use bounded integer arithmetic, canonicalize trusted keys, update shared state atomically, separate budgets by operation and tenant, return stable HTTP behavior, test real storage and concurrency, observe outcomes without leaking identity, and roll out policy versions with explicit degraded paths.

## References

- [RFC 6585 — Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [PHP Manual: `hrtime()`](https://www.php.net/manual/en/function.hrtime.php)
- [Chapter 141 — Idempotency](../09-http-and-application-development/141-idempotency.md)
- [Chapter 246 — Backpressure](../16-distributed-systems/246-backpressure.md)
- [Chapter 247 — Circuit Breakers](../16-distributed-systems/247-circuit-breakers.md)
- [Chapter 248 — Bulkheads](../16-distributed-systems/248-bulkheads.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 280 — Tennis Reservation Service](./280-tennis-reservation-service.md)
