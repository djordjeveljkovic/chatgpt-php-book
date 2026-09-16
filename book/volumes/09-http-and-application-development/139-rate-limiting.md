---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 139
title: Rate Limiting
slug: rate-limiting
status: complete
summary: ../../_ai/chapter-summaries/139-rate-limiting-summary.md
---

# Chapter 139 — Rate Limiting

## Why This Matters

Rate limits protect capacity and make access policy visible. They can slow abuse, prevent one client from exhausting a database pool, and provide fairer access during a traffic spike. A limit is not authentication, authorization, or a complete denial-of-service defense; it must be placed at the boundary where the relevant identity and cost are known.

## Mental Model

```text
identify client and operation
        ↓
atomically consume allowance
        ↓
allow or reject with Retry-After
```

Define the key (IP, account, API key, tenant, or operation), window, quota, burst policy, and failure behavior. A distributed application needs shared or consistently partitioned state; a PHP static counter protects one worker only.

## Algorithms and Headers

Fixed windows are simple but allow bursts at boundaries. Sliding windows are smoother but need more state. Token buckets permit controlled bursts while enforcing an average rate. A limiter should return a stable policy response such as `429 Too Many Requests`, with a bounded `Retry-After` value when known.

Limit expensive operations separately from cheap reads. A password attempt, export, upload, and search query should not necessarily share one bucket. Do not expose enough detail for an attacker to enumerate another user's limit key.

## Atomic State

Redis `INCR`/expiry scripts, a database row under an atomic update, or an edge proxy can implement shared counters. The increment and expiry decision must be atomic enough that concurrent requests cannot all observe the same remaining allowance. Define what happens if the limiter store is unavailable: fail closed for sensitive operations, fail open for low-risk reads, or use a bounded local fallback.

## PHP Boundary

Keep rate-limit policy separate from the controller:

```php
<?php

declare(strict_types=1);

final readonly class LimitDecision
{
    public function __construct(
        public bool $allowed,
        public int $remaining,
        public int $retryAfterSeconds,
    ) {
    }
}

function applyHeaders(LimitDecision $decision): array
{
    return [
        'X-RateLimit-Remaining' => (string) max(0, $decision->remaining),
        ...(!$decision->allowed
            ? ['Retry-After' => (string) $decision->retryAfterSeconds]
            : []),
    ];
}
```

The decision object does not implement the counter; it keeps transport representation separate from the atomic store adapter.

## Operations and Testing

Monitor allowed/rejected rates, key cardinality, store latency, fallback use, and false-positive complaints. Test boundary bursts, concurrent requests, clock skew, store failure, authenticated versus anonymous keys, and trusted proxy configuration. Load-test the limiter itself; a rate limiter that becomes a bottleneck is a new outage source.

## Common Mistakes

- Counting only in one PHP worker.
- Using an untrusted `X-Forwarded-For` as identity.
- Limiting all operations with one bucket.
- Making a distributed counter non-atomic.
- Returning a retry interval that causes synchronized storms.
- Treating rate limiting as authorization or DDoS protection.

## Senior Engineer Thinking

Rate limiting is capacity policy. Select the identity and cost model, place enforcement before expensive work, choose failure behavior deliberately, and expose enough response metadata for cooperative clients without leaking policy details.

## Exercises

1. Compare fixed-window and token-bucket behavior at a window boundary.
2. Design separate limits for login, search, and export.
3. Define fail-open and fail-closed behavior for a limiter-store outage.

## Review Questions

1. Why does a PHP-local counter fail across workers?
2. Which operations deserve separate limits?
3. What must be atomic in a distributed limiter?
4. Why is a limiter not a complete DDoS defense?

## Summary

Rate limiting protects capacity and fairness when keyed to the right actor and operation. Use an atomic shared algorithm, return clear `429` behavior, define store-failure policy, and observe both rejected traffic and limiter health.

## References

- [RFC 6585: 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585)
- [RFC 9110: `Retry-After`](https://www.rfc-editor.org/rfc/rfc9110)
- [OWASP: API4 Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/)
