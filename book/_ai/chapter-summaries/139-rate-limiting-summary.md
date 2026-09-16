# AI Summary — Chapter 139 — Rate Limiting

- Status: complete
- Volume: Volume 9 — HTTP AND APPLICATION DEVELOPMENT
- Last updated: 2026-09-16

## Written material

Explains rate-limit keys and algorithms, atomic shared state, operation-specific quotas, `429`/`Retry-After`, failure policy, observability, testing, exercises, and review questions.

## Concepts already explained

Rate limiting is capacity policy, not authentication or authorization. Fixed windows, sliding windows, and token buckets trade simplicity, burst behavior, and state. Distributed counters require atomic operations and deliberate fail-open/closed behavior.

## Terminology established

Rate limit, fixed window, sliding window, token bucket, burst, allowance, limiter key, `Retry-After`.

## Examples used

A typed PHP limit decision and response-header mapping; shared-store, failure, and operation-specific policies.

## Cross-references

- [Chapter 134 — APIs](../../volumes/09-http-and-application-development/134-apis.md)
- [Chapter 141 — Idempotency](../../volumes/09-http-and-application-development/141-idempotency.md)

## Exact next section

Chapter 140 — API Versioning: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated HTTP/security proofread. HTTP status and retry guidance links to RFC 6585/9110 and OWASP.
