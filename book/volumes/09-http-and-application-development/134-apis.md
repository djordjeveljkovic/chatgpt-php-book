---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 134
title: APIs
slug: apis
status: complete
summary: ../../_ai/chapter-summaries/134-apis-summary.md
---

# Chapter 134 — APIs

## Why This Matters

An API is a contract between independently deployed callers and a service. The contract includes more than a route and JSON shape: it includes authentication, validation, status codes, error structure, pagination, idempotency, compatibility, rate limits, and timeout behavior. A response that looks reasonable in a browser can still be unusable for a client that must retry or distinguish a missing resource from an authorization failure.

## Mental Model

```text
request
  ↓ parse and authenticate
  ↓ authorize and validate
  ↓ execute domain command/query
  ↓ map result or problem
response with stable contract
```

Keep transport DTOs separate from domain entities. The API boundary chooses which fields are public and how errors are represented; database columns and internal exception messages are not automatically public API.

## Resource and Command Boundaries

Choose resource names and actions from the domain. `POST /reservations` can create a reservation; `POST /reservations/{id}/cancel` may represent an explicit command when cancellation has rules that do not map cleanly to a generic update. Document allowed transitions and return the resulting representation or an operation identifier.

Validate JSON syntax, content type, size, required fields, unknown-field policy, and relationships between values. Keep validation errors machine-readable:

```json
{
  "type": "https://api.example.test/problems/validation",
  "title": "Request validation failed",
  "status": 422,
  "errors": {
    "starts_at": ["Must be before ends_at"]
  }
}
```

Use a consistent problem format and do not include stack traces, SQL, tokens, or internal identifiers in production responses. Correlate a safe request ID in logs and, when useful, a response header.

## Status Codes and Caching

Use status codes to describe the transport outcome, then use the body for domain details. `400` can indicate malformed syntax, `401` missing or invalid credentials, `403` an authenticated actor without permission, `404` an absent or intentionally undisclosed resource, `409` a state conflict, `422` valid syntax with invalid domain fields, and `429` a rate limit. Choose a policy and apply it consistently; clients should not need to guess whether every failure is a `200` with an error field.

For safe reads, define cache behavior with `Cache-Control`, validators such as `ETag`, and authorization rules. Never let a shared cache store a personalized response unless the cache key and directives make that separation explicit. A cache hit is not a substitute for current authorization when the resource is private.

## Authentication, Authorization, and Input

Authentication establishes an actor; authorization checks the actor against the resource and action. Validate both at the service boundary, not only in a controller or gateway. Avoid trusting client-supplied ownership, role, or tenant fields.

Reject unknown fields when accepting sensitive commands unless forward compatibility requires ignoring them. A permissive mass-assignment mapper can let a caller set `is_admin`, `status`, or another field that was not intended to be writable. Map allow-listed fields explicitly.

## Retries and Idempotency

A client cannot always know whether a timeout happened before or after the server committed. Mutating endpoints need a documented retry policy. Use an idempotency key for operations where duplicate effects are harmful, persist the key with the result, and return the original result for an identical retry. A reused key with a different payload should be a conflict.

Set server and client timeouts below infrastructure limits, propagate cancellation where possible, and make downstream calls bounded. Do not blindly retry a `POST` that creates a charge or sends an email without a durable idempotency design.

## Observability and Testing

Measure request rate, latency distributions, status counts, payload sizes, dependency time, and retry counts. Redact authorization headers, cookies, secrets, and sensitive fields. Distributed tracing and a correlation ID help connect an API request to database and queue work.

Contract tests should verify JSON shape, status codes, headers, authentication failures, validation errors, idempotent retries, pagination cursors, and compatibility with representative clients. Integration tests must exercise the real serialization and authorization boundary; controller unit tests alone can miss middleware and content-negotiation behavior.

## Common Mistakes

- Treating internal entities or exception messages as a stable public contract.
- Returning `200` for every outcome.
- Confusing authentication with authorization.
- Logging bearer tokens or personal data.
- Retrying non-idempotent writes after a timeout.
- Allowing mass assignment of fields not intended for clients.
- Omitting pagination and size limits from collection endpoints.
- Changing error shapes without a compatibility plan.

## Senior Engineer Thinking

Design an API around client decisions and failure recovery. For every endpoint, specify who may call it, what happens when the request is repeated, which state transitions are legal, how a client learns what failed, and how the contract evolves. The API should make the safe path easy for callers under latency, retry, and partial-failure conditions.

## Exercises

1. Define a problem response for invalid reservation intervals and a separate response for a conflicting reservation.
2. Design an idempotency-key contract for a payment-like command.
3. Write an API error policy distinguishing `401`, `403`, `404`, `409`, and `422`.
4. List the fields that must be redacted from request logs for your API.

## Review Questions

1. What belongs in an API contract beyond the JSON body?
2. When should a service return `409` rather than `422`?
3. Why can a timeout make a client unsure whether a write committed?
4. Which values should never be logged by default?

## Summary

An API contract includes behavior under success, validation failure, authorization failure, retries, caching, and evolution. Separate transport models from domain objects, use consistent statuses and problem responses, authenticate and authorize explicitly, bound dependencies, and make harmful mutations idempotent.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [OWASP: API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
