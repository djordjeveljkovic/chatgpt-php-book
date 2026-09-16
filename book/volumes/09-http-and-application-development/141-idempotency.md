---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 141
title: Idempotency
slug: idempotency
status: complete
summary: ../../_ai/chapter-summaries/141-idempotency-summary.md
---

# Chapter 141 — Idempotency

## Why This Matters

A client can lose the response without losing the request. The server may have charged a card, created an order, or enqueued an email just before a timeout. If the client retries and the operation is not designed for repetition, one user action can produce two effects.

Idempotency makes repeated processing of the same logical operation converge on one intended result. It is a property of an operation and its state transition, not a promise that every retry has the same transport bytes or latency.

## Safe, Idempotent, and Non-Idempotent

HTTP defines `GET`, `HEAD`, `OPTIONS`, `PUT`, and `DELETE` as idempotent in their intended effect, while `POST` is not generally idempotent. A `PUT` that replaces a resource with the same representation can be retried; a `POST` that creates a new subordinate resource can create another resource each time. Server-side logging may still happen on each request, so idempotency describes the requested state change rather than every side effect.

Application operations need their own analysis. Sending a notification, reserving inventory, charging money, publishing an event, and issuing a refund each have external effects. A database unique constraint can prevent duplicate rows, but it cannot by itself prevent a downstream provider from receiving two calls after a process crash.

## Idempotency Keys

For a retryable command, the client sends a high-entropy key scoped to the operation and authenticated account:

```http
POST /payments HTTP/1.1
Idempotency-Key: 7b7a9d63-1ec0-4d21-b4b8-b3cf90c2f9a1
Content-Type: application/json

{"order_id":"order-42","amount":1999,"currency":"EUR"}
```

The server stores the key, account or tenant, a fingerprint of the canonical request, processing state, and the final response or durable result. A retry with the same key and equivalent payload returns the stored result. Reusing a key with a different payload is a conflict; otherwise a client bug or attacker could attach a new command to an old result.

Keys must be scoped. The same string supplied by two accounts must not share a result, and a key for one endpoint should not accidentally replay a result for another. Set a retention period long enough to cover the client's retry window and business duplicate risk. Expiring a key while a client can still retry recreates the duplicate problem, so retention is a domain decision rather than an arbitrary cache TTL.

## A Durable Claim

The first request must claim the key atomically. A check followed by an insert has a race:

```text
request A: SELECT no row
request B: SELECT no row
request A: perform charge
request B: perform charge
```

Use a unique constraint on `(account_id, operation, idempotency_key)` and a transaction or atomic insert. A concurrent request can receive a conflict or a response that tells it to retry while the first operation is still running. Do not hold a database transaction open while waiting indefinitely for a remote provider.

```php
<?php

declare(strict_types=1);

final readonly class IdempotencyRecord
{
    public function __construct(
        public int $accountId,
        public string $operation,
        public string $key,
        public string $requestHash,
        public string $state,
        public ?string $responseJson,
    ) {
    }
}

function canonicalize(mixed $value): mixed
{
    if (!is_array($value)) {
        return $value;
    }

    if (array_is_list($value)) {
        return array_map(canonicalize(...), $value);
    }

    $keys = array_keys($value);
    sort($keys, SORT_STRING);
    $normalized = [];
    foreach ($keys as $key) {
        $normalized[$key] = canonicalize($value[$key]);
    }

    return $normalized;
}

function requestHash(array $payload): string
{
    $canonicalPayload = canonicalize($payload);
    return hash('sha256', (string) json_encode(
        $canonicalPayload,
        JSON_THROW_ON_ERROR | JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE,
    ));
}
```

Canonicalization is part of the contract. If object key order, numeric representation, or omitted defaults can vary between equivalent requests, define a canonical serializer before hashing. Hashing raw request bytes may incorrectly treat semantically equal JSON as different; decoding and re-encoding without a defined key order can also produce unstable fingerprints. Never use the client key as an authorization credential by itself.

## State and Recovery

A useful record state is `processing`, `succeeded`, or `failed` with a retry policy. Store enough information to replay a safe response, such as status, selected headers, and body, or store a durable result from which the response can be reconstructed. Do not cache an arbitrary exception message or a response containing a secret.

If a worker crashes after claiming a key, a permanent `processing` record can block the customer forever. Include a lease or heartbeat, detect stale work, and make recovery inspect the downstream operation before attempting it again. A timeout from a payment provider is ambiguous: the provider may have committed. Query the provider by its own idempotency key or operation identifier before retrying a charge.

For local database effects and an outbox event, commit both in one transaction, then deliver the outbox asynchronously. The outbox prevents a crash between the database commit and event publish, but consumers still need their own deduplication or idempotent handling. Exactly-once behavior across arbitrary systems is not obtained by adding a flag to a PHP class.

## HTTP Responses and Security

Define status behavior for a first request, an in-progress duplicate, a completed duplicate, a payload mismatch, and an expired key. `409 Conflict` or `422 Unprocessable Content` may be appropriate for a mismatch according to the API policy; an in-progress duplicate may return `409` or `202` with a retry hint. The key point is consistency and documentation.

Bind a record to the authenticated subject and operation. Validate authorization again when replaying a result, especially if access can be revoked between attempts. Limit key length, avoid logging raw keys when they can correlate sensitive actions, and rate-limit claims so an attacker cannot fill the table indefinitely. A key should be unguessable enough to prevent accidental collisions, but it must never substitute for authentication.

## Failure and Threat Analysis

* **Check-then-act race:** enforce uniqueness in the database and handle duplicate-claim outcomes.
* **Crash after side effect:** use a provider-side idempotency key or reconciliation query.
* **Stale processing row:** lease work and define recovery, rather than blocking forever.
* **Payload mismatch:** compare a canonical request fingerprint and reject reuse.
* **Cross-tenant replay:** include authenticated account and operation in the uniqueness scope.
* **Unbounded storage:** apply retention and cleanup only after the duplicate-risk window.
* **Duplicate downstream delivery:** design consumers to tolerate repeated event IDs.
* **Sensitive replay:** store and return only the minimum response data needed by the contract.

## Testing Idempotent Commands

Test a successful first request followed by the same request, a concurrent duplicate, a changed payload with the same key, an expired key, and a retry after a simulated process crash. Assert that the domain effect occurs once and that the duplicate receives the documented result. Use a fake provider that records calls and can commit a side effect before throwing a timeout.

Test authorization and tenant scoping on replay. For asynchronous work, deliver the same event twice and verify the consumer's final state. Include database constraint tests against the real database engine because an in-memory fake may not reproduce uniqueness, isolation, or deadlock behavior.

## Exercises

1. Design an idempotency table with a unique scope, request fingerprint, processing lease, result, and expiry. Explain each index.
2. Implement a create-order command that uses an idempotency key and an outbox event. Identify the crash points and recovery actions.
3. Write tests for two concurrent requests using one key and for the same key used with a different payload.
4. Model a payment-provider timeout. Decide how your application discovers whether the provider committed before retrying.

## Review Questions

1. What does idempotency guarantee, and what does it not guarantee?
2. Why is a check followed by an insert unsafe under concurrency?
3. Why must an idempotency record include the authenticated subject and operation?
4. How should a system recover a stale `processing` record?
5. Why does an outbox still require idempotent consumers?
6. What is the danger of expiring keys too early?

## Summary

Retries are inevitable when clients and networks fail. Make harmful commands idempotent with an atomically claimed, scoped key, a canonical request fingerprint, a durable result, and an explicit recovery state. Coordinate database effects with an outbox, use provider-side deduplication for external effects, and test races, crashes, mismatches, replay authorization, and retention.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Stripe: Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [IETF HTTPAPI: Idempotency-Key HTTP Header Field](https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-07.html)
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
