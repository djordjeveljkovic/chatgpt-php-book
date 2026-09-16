---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 242
title: Idempotency
slug: idempotency
status: complete
summary: ../../_ai/chapter-summaries/242-idempotency-summary.md
---

# Chapter 242 — Idempotency

## Why This Matters

An idempotent operation has the same intended effect when the same request is applied more than once. Networks, clients, queues, and workers can repeat an attempt, so effecting operations need a way to converge rather than create a second charge, reservation, or email.

Idempotency is not the same as “the endpoint returns the same JSON.” A read may return the same representation while a hidden side effect repeats. Define what effect is protected, which requests are considered the same, how long the identity is retained, and what happens when the same key is reused with different input.

## Identity and Scope

An idempotency key identifies an intended operation, not merely a TCP request. Its scope should include the authority and actor or tenant where needed:

```text
tenant + operation type + client key
```

Store a hash of the normalized request alongside the key. A repeated key with a different amount or recipient is a conflict, not a new operation. Never let one tenant query or replay another tenant's key.

Choose retention from the duplicate window: client retry duration, queue redelivery, provider recovery, and reconciliation time. A key that expires while an old request can still be replayed does not protect the full risk window.

## State Machine

An idempotency record can track progress:

```text
absent → processing → completed
             ↓             ↑
           failed ── retry ─┘
```

The exact states depend on the operation. `processing` must have a lease or recovery policy; a crashed owner must not leave the key permanently blocked. `completed` should retain the result or a durable reference so a duplicate can receive the original outcome. A permanent failure may be replayable only if the request is corrected under a new key.

## Database Enforcement

The idempotency record and the business transition should share an authority when they must be atomic. A unique constraint on `(tenant_id, operation_type, client_key)` prevents two concurrent inserts from both becoming the owner. The transaction then records the request hash and state before applying the invariant.

Do not implement “check then insert” without a constraint:

```text
request A: SELECT no record
request B: SELECT no record
request A: INSERT
request B: INSERT
```

The database must arbitrate the race. On a duplicate-key result, load the existing record, compare the request hash, and return or wait according to its state.

## A Narrow Service Example

The storage and transaction interfaces are illustrative:

~~~php
<?php

declare(strict_types=1);

final readonly class IdempotencyKey
{
    public function __construct(
        public int $tenantId,
        public string $operation,
        public string $clientKey,
        public string $requestHash,
    ) {
        if ($tenantId <= 0 || $operation === '' || $clientKey === '' || $requestHash === '') {
            throw new InvalidArgumentException('Invalid idempotency identity');
        }
    }
}

interface IdempotencyStore
{
    public function claim(IdempotencyKey $key): ClaimResult;
    public function complete(IdempotencyKey $key, string $result): void;
}

function executeOnce(
    IdempotencyStore $store,
    IdempotencyKey $key,
    callable $effect,
): string {
    $claim = $store->claim($key);

    if ($claim->isCompleted()) {
        return $claim->result();
    }

    if (!$claim->isOwner()) {
        throw new OperationInProgress('Another request owns this key');
    }

    $result = $effect();
    $store->complete($key, $result);

    return $result;
}
~~~

The crash window between `$effect()` and `complete()` remains. If the effect is outside the store's transaction, the effect itself needs an idempotency key or a reconciliation path. A local record does not magically make a remote provider atomic.

## Request Hashes and Results

Normalize only fields whose semantic representation is stable. Hash the tenant, operation, relevant parameters, and version—not an arbitrary serialized object whose field order or formatting can change. Use a cryptographic hash for comparison and do not treat a hash as a secret.

Store the minimum response needed to replay the contract: status, result reference, safe response body or a lookup ID, and schema version. Do not store a bearer token or sensitive payment data merely because it appeared in the original request.

If the original result is no longer available, return a stable “completed; retrieve by operation ID” response rather than executing again. Expiration should be explicit and documented.

## Idempotency and Queues

At-least-once delivery means a handler may receive the same message again. Put the message or domain operation identity in a durable record and make the handler converge. A queue message ID can identify delivery; a domain operation ID identifies the intended effect. They are not always interchangeable because the same business command can be wrapped in multiple messages.

An acknowledgment after the side effect still leaves a crash window. The handler can safely replay only when the side effect is protected by a unique constraint, an idempotency key, or a state transition that rejects an already-applied version.

## Idempotency and HTTP

The API contract should define when a client may reuse a key, how long it is valid, whether concurrent reuse waits or returns conflict, and how a request-hash mismatch is reported. Require authentication before looking up a record and scope it to the authorized tenant.

Do not accept a client-provided key as proof of identity or permission. It is an operation identifier supplied by an untrusted caller. Authorization is evaluated for every request; a replay should not bypass a permission change.

## Concurrency

Two requests with the same key may arrive concurrently. Options include:

* one atomically claims the key and the other waits with a deadline;
* the second receives an in-progress response and polls;
* the second receives a conflict and retries later;
* both converge through a database state transition.

Choose based on the user contract. Waiting while holding a PHP-FPM worker or database transaction can reduce capacity. A short claim lease plus a recovery job is safer than an unbounded lock.

## Testing

Test sequential duplicate requests, concurrent claims, same key with changed input, retries after timeout, crash after the effect but before completion, expired records, replay after authorization changes, and a late duplicate after recovery. Assert the number of real effects, not just the number of responses.

Use a database integration test for the unique constraint and transaction race. Use a provider sandbox or fake that records idempotency keys for the remote boundary. Test key retention and cleanup so an unbounded idempotency table does not become a performance problem.

## Common Mistakes

* Generating a fresh key for every retry.
* Treating a transport request ID as a business operation identity.
* Relying on an application-level check without a unique constraint.
* Returning the first result without comparing the new request hash.
* Leaving a processing record forever after a worker crash.
* Storing secrets in replayable idempotency responses.
* Letting replay bypass current authentication or authorization.
* Assuming a local idempotency table protects a remote side effect.

## Senior Engineer Thinking

Ask which duplicate is dangerous, which authority can enforce uniqueness, and what evidence remains after each crash window. Idempotency is a contract among caller, owner, storage, queue, and provider; a random header added to one endpoint is not the whole design.

## Exercises

1. Design an idempotency record for a payment operation. Include key scope, request hash, result, retention, lease, and authorization.
2. Reproduce the check-then-insert race with two concurrent database transactions and fix it with a unique constraint.
3. Model a crash after a provider charge and before the local completion record. Define the reconciliation process.
4. Decide whether a queue message ID or a domain operation ID should protect each of three handlers.

## Review Questions

* What does idempotency protect, and what does it not protect?
* Why must a repeated key be compared with a request hash?
* Which authority should enforce concurrent uniqueness?
* Why can processing state need a lease?
* How does a domain operation ID differ from a delivery ID?
* Why must authorization still run on a replay?

## Summary

Idempotency makes repeated attempts converge on one intended effect. Scope an operation key, bind it to a normalized request hash, enforce uniqueness at the owning authority, store a recoverable result, handle processing leases and unknown completion, and protect replays with current authorization. Test races and crash windows, not only sequential duplicates.

## References

- [Stripe: Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [RFC 9110: Idempotent methods](https://www.rfc-editor.org/rfc/rfc9110#section-9.2.2)
- [Chapter 141 — Idempotency](../../volumes/09-http-and-application-development/141-idempotency.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
