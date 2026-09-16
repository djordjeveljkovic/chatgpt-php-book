---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 140
title: API Versioning
slug: api-versioning
status: complete
summary: ../../_ai/chapter-summaries/140-api-versioning-summary.md
---

# Chapter 140 — API Versioning

## Why This Matters

An API is a promise made to independently deployed clients. A server can improve its internal model while an old mobile application, partner integration, or queued message continues to send yesterday's request. Versioning gives incompatible contracts a controlled lifetime and gives clients a migration path.

Versioning is not a license to duplicate an application forever. It is a decision about compatibility: which changes are safe, which require a new representation or operation, how long an old contract remains available, and how the team knows when it can be removed.

## What a Version Actually Covers

Treat these as separate compatibility surfaces:

* **Representation:** field names, types, nullability, enum values, pagination and error shapes.
* **Behavior:** authorization rules, state transitions, ordering, default filters, retry and idempotency behavior.
* **Transport:** URLs, media types, headers, status codes and authentication schemes.
* **Operational contract:** limits, timeout expectations, rate limits and deprecation dates.

Adding an optional response field is often compatible. Removing a field, changing its type, changing an enum's meaning, or making a previously accepted request invalid can break clients. A behavior change can be breaking even when the JSON schema is unchanged. Write compatibility rules for the API rather than relying on intuition.

## Choosing a Versioning Boundary

Common choices include a path such as `/v2/orders`, a media type such as `application/vnd.example.order+json;v=2`, or a version negotiated through a request header. Each can work. A path is visible and easy to route and document; media-type negotiation keeps one resource URI but is less obvious in logs and manual requests. A custom header can be useful inside a controlled ecosystem but is easy for clients and caches to omit.

Do not send two different representations from the same cache key. If representation negotiation affects the response, send `Vary: Accept` (or the relevant version header) and configure the cache accordingly. A gateway may select a version, but the application still has to enforce the contract and authorization rules for the selected version.

```php
<?php

declare(strict_types=1);

enum ApiVersion: int
{
    case V1 = 1;
    case V2 = 2;
}

function requestedVersion(string $path): ApiVersion
{
    if (str_starts_with($path, '/v2/')) {
        return ApiVersion::V2;
    }

    if (str_starts_with($path, '/v1/')) {
        return ApiVersion::V1;
    }

    throw new InvalidArgumentException('Unsupported API version');
}
```

The version selector should be explicit and tested. Do not silently route an unknown version to the newest behavior; that turns a typo into an unpredictable contract change.

## Adapters at the Boundary

Keep version-specific request and response objects at the transport boundary. The domain model should express business concepts without carrying every historical spelling. An adapter maps each public contract to a stable command and maps the result back:

```php
<?php

/** @return array<string, mixed> */
function orderResponse(ApiVersion $version, Order $order): array
{
    return match ($version) {
        ApiVersion::V1 => [
            'id' => (string) $order->id,
            'total_cents' => $order->totalCents,
        ],
        ApiVersion::V2 => [
            'id' => (string) $order->id,
            'total' => [
                'amount' => $order->totalCents,
                'currency' => $order->currency,
            ],
        ],
    };
}
```

An adapter is valuable because it makes the compatibility decision visible. It is also a place to preserve old defaults, translate old enum values, and emit metrics by version. Do not solve versioning by scattering `if ($version === ...)` through domain services; that makes it difficult to prove that both contracts enforce the same invariant.

Request adapters should allow-list fields. A version that historically accepted `name` should not accidentally gain write access to an internal `status` column because a generic hydrator was reused. Unknown-field policy must be documented: rejecting unknown fields catches client mistakes, while ignoring them can help additive evolution. Sensitive commands usually benefit from rejection.

## Safe Evolution and Deprecation

Prefer additive changes: add a field, add an endpoint, or add a new optional capability while preserving old behavior. When introducing a replacement field, document precedence and a deadline. A server may return both fields temporarily, but the values must have defined semantics and tests.

Deprecation is a product and operations process. Publish the support window, migration guide, owner, and removal date. Use documentation, client communication, dashboards, and version-specific metrics to find remaining callers. HTTP provides `Deprecation` and `Sunset` response headers for communicating lifecycle information, but clients cannot act on headers they never see; document them and test them. Never remove a version merely because the new version exists if contractual or regulatory retention requires continued support.

Record a safe client identifier and version in metrics. Do not put access tokens, personal data, or raw request bodies in version telemetry. A client that sends no explicit version should receive a deliberate default with a documented policy; changing that default silently is equivalent to a breaking change.

## Database and Deployment Strategy

An API migration often crosses application and database deployments. Use an expand-and-contract sequence:

1. Add the new nullable column or table shape.
2. Deploy code that can read both forms and writes the compatible form.
3. Backfill and verify data, with a resumable job and measured load.
4. Switch new-version traffic to the new representation.
5. Remove old writes, then old reads, after the support window.

Deploying a new controller that assumes a column added in a later migration creates a rollback hazard. Keep the intermediate schema readable by both the old and new application versions. For asynchronous events, version the event schema or publish an explicitly compatible envelope; a queue can contain old messages long after a deployment finishes.

## Failure and Threat Analysis

* **Unknown version:** return a documented client error and do not guess a behavior.
* **Mixed-version authorization:** run the same domain authorization policy after request adaptation; an old endpoint must not become a bypass.
* **Cache confusion:** include the negotiated version in the cache key and `Vary` policy.
* **Dual-write drift:** compare or reconcile old and new forms while both are written.
* **Forgotten clients:** monitor traffic by version and identify SDK, partner, and job callers.
* **Replay across contracts:** preserve idempotency semantics and resource identifiers when a request is retried against a supported version.
* **Sensitive migration logs:** redact payloads and use aggregate metrics for adoption.

## Testing Versioned Contracts

Contract tests should assert requests, responses, headers, error shapes, authorization, and status codes for every supported version. Test an old client against a new deployment and a new client against the migration database shape. Use fixture data that covers nulls, enum additions, large values, and boundary timestamps.

Schema compatibility tools can check additive JSON changes, but they cannot prove behavior or authorization compatibility. Add end-to-end tests for deprecation headers, cache variation, pagination cursors, conditional requests, and idempotency. Test the removal procedure in a staging environment so an old route cannot be accidentally left enabled without an owner.

## Exercises

1. Define compatible and breaking changes for a `GET /orders/{id}` response. Include fields, enum values, nullability, and error shapes.
2. Implement a v1-to-domain request adapter and a v2-to-domain request adapter. Prove that both enforce the same authorization and state-transition rules.
3. Design an expand-and-contract migration for renaming an order's `total_cents` field while v1 and v2 run simultaneously.
4. Create a deprecation dashboard showing calls by version, client, status, and endpoint without recording credentials or personal data.

## Review Questions

1. Which changes are breaking even when a JSON parser still succeeds?
2. What is the trade-off between path and media-type versioning?
3. Why should version-specific mapping stay near the transport boundary?
4. How can a cache return the wrong API representation?
5. Why must database changes support both old and new application versions during deployment?
6. What evidence should be collected before removing an API version?

## Summary

API versioning manages compatibility across representations, behavior, transport, and operations. Choose a visible boundary, adapt versioned DTOs at the edge, prefer additive evolution, publish deprecation policy, and use expand-and-contract database changes. Test contracts and authorization for each supported version, and make cache, retry, telemetry, and removal behavior explicit.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [RFC 8594: The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594)
- [RFC 9745: The Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745)
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
