---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 236
title: Network Boundaries
slug: network-boundaries
status: complete
summary: ../../_ai/chapter-summaries/236-network-boundaries-summary.md
---

# Chapter 236 — Network Boundaries

## Why This Matters

A network call crosses an ownership and failure boundary. The caller and receiver execute in different processes, may run on different machines, and do not share memory or a clock. A PHP method call either returns or throws within one process; an HTTP call can be delayed, duplicated, partially completed, rejected by a proxy, or lost after the receiver commits its work.

Treat every network interaction as a contract between independent parties. Define the message shape, authentication, deadlines, response meanings, retry rules, idempotency behavior, observability, and compatibility window before choosing a client library. A convenient SDK does not remove these design obligations.

## Mental Model

The boundary has more steps than the application call suggests:

```text
PHP code
  ↓ serialize and validate
client connection → DNS → TCP/TLS → proxy/load balancer
  ↓                                                 ↓
request bytes                                  server process
                                                   ↓
                                      deserialize, authorize, execute
                                                   ↓
response bytes ← proxy/load balancer ← server response
```

Any step can fail independently. A successful TCP connection does not mean the business operation succeeded. A response status does not prove the client received every byte. A client timeout does not prove that the server stopped working.

Separate these questions:

* Who owns the data and invariant?
* Is the interaction synchronous, asynchronous, or both?
* What is the request identity and duplicate policy?
* Which failures are safe to retry?
* What can the caller observe if the response is lost?
* How are versions deployed while old and new clients overlap?

## Contract at the Boundary

A useful contract specifies more than fields:

* request and response schema, including unknown-field behavior;
* authentication and authorization requirements;
* maximum body, item, and header sizes;
* status and error categories that clients can act on;
* deadline and rate-limit behavior;
* idempotency key or operation identity where effects are possible;
* correlation and trace context without sensitive payloads;
* compatibility and deprecation policy.

Keep domain objects inside the owning service. Map them to a boundary representation deliberately. Exposing an ORM row or internal exception structure couples consumers to storage and makes harmless refactoring an API change.

## Synchronous and Asynchronous Work

Synchronous calls are appropriate when the caller needs a bounded answer before continuing. They consume caller resources while waiting and couple availability to the receiver. Use a finite deadline and a clear fallback or error.

Asynchronous work lets the caller receive an accepted or pending result and moves completion to a queue or workflow. It improves burst handling but requires durable state, status lookup or notifications, duplicate handling, and a user-visible meaning for pending and failed outcomes.

Do not turn a synchronous payment authorization into a fire-and-forget request merely to improve latency. Change the product contract: acknowledge that the request is pending, persist the operation, and provide reconciliation.

## Serialization Is a Boundary

Serialization converts typed internal values into bytes. Validate both sides of the conversion. JSON is a representation, not a type system: numbers can lose precision across languages, absent and `null` fields can mean different things, and object shape can evolve.

Use explicit DTOs or arrays with a documented shape. Reject invalid or oversized input before expensive processing. Never deserialize untrusted data into executable object graphs without a narrowly defined, authenticated format. See the serialization and security discussions in [Chapter 39](../../volumes/03-php-object-model/039-serialization.md) and [Chapter 151](../../volumes/10-security/151-deserialization.md).

For a response, distinguish:

```text
field absent     → not supplied or not supported
field null       → explicitly empty, if the contract permits it
field value      → present and valid
```

Choose one meaning per field and test it. A client that silently treats an absent permission list as “all permissions” has converted a representation detail into a security defect.

## A Small Client Boundary

Hide transport details behind a narrow port, but keep failure information useful:

~~~php
<?php

declare(strict_types=1);

final readonly class CreateInvoiceRequest
{
    public function __construct(
        public string $requestId,
        public int $customerId,
        public int $amountCents,
    ) {
        if ($requestId === '' || $customerId <= 0 || $amountCents <= 0) {
            throw new InvalidArgumentException('Invalid invoice request');
        }
    }
}

interface InvoiceGateway
{
    public function create(CreateInvoiceRequest $request, int $deadlineNs): string;
}

function createInvoice(InvoiceGateway $gateway, CreateInvoiceRequest $request): string
{
    $deadlineNs = hrtime(true) + 800_000_000;

    return $gateway->create($request, $deadlineNs);
}
~~~

The port carries a request identity and an absolute deadline. The concrete adapter maps its transport's timeout and error types to the application's categories. The caller still needs a policy for an ambiguous result: it may query the operation by `requestId` rather than blindly create another invoice.

## Bad and Better Boundaries

This boundary is too vague:

~~~php
function callService(array $data): array
{
    return $http->post('/whatever', $data);
}
~~~

It hides the endpoint, schema, deadline, authentication, response validation, and duplicate policy. The type `array` says almost nothing about the contract.

A better boundary names the operation and validates its result:

~~~php
final readonly class InvoiceCreated
{
    public function __construct(public string $operationId)
    {
        if ($operationId === '') {
            throw new UnexpectedValueException('Missing operation identity');
        }
    }
}

interface BillingClient
{
    /** @throws BillingUnavailable */
    public function createInvoice(CreateInvoiceRequest $request, int $deadlineNs): InvoiceCreated;
}
~~~

The docblock does not make the transport reliable, but it makes the intended failure category visible to callers and tests. Validate status, content type, body size, schema, and authorization context in the adapter.

## What PHP Does

PHP code usually blocks while a synchronous client waits for the transport. During that wait, the PHP-FPM worker remains occupied even though it may use little CPU. In a long-running worker, a connection, stream, response buffer, or tracing context can survive longer than intended if the adapter does not close or reset it.

Use the client library's documented timeout and cancellation behavior. A PHP exception is a local observation; it does not cancel work already accepted by a remote service unless the protocol provides cancellation. Always release response bodies and reset request-scoped context in long-running processes.

## Compatibility and Ownership

Prefer additive changes: add an optional response field, deploy consumers that ignore it safely, then make it required only after all consumers support it. Removing a field, changing units, changing an enum meaning, or tightening validation is a compatibility event.

An API version is not only a URL suffix. It can be a media type, schema version, message version, or capability negotiation. Record which clients use which contract and define removal evidence. A service should own the meaning of its response; consumers should not infer private database state from accidental fields.

## Security

Network boundaries are trust boundaries. Authenticate the peer, authorize the operation for the caller and tenant, protect credentials, validate redirects and destinations, and use encryption appropriate to the transport. Do not forward a user's bearer token to an unrelated downstream service by default.

Bound request size and response size. An authenticated peer can still send a huge or malicious payload. Redact tokens, payment data, and personal data from logs, traces, exceptions, and retry messages. Treat correlation IDs as identifiers, not authorization.

## Testing

Test the adapter against a contract fixture or compatible test server. Cover valid responses, malformed JSON, wrong content type, oversized bodies, unknown fields, authentication failure, authorization failure, rate limiting, timeouts, connection failure, and a response lost after server-side completion.

Test the application port with a fake that models the business outcome, and test the adapter separately for transport behavior. Do not make every unit test depend on a live network. Run a smaller integration suite against the actual protocol and deployment configuration.

## Common Mistakes

* Treating a network call as a local method call.
* Returning raw transport responses throughout the domain.
* Omitting a deadline because the library has a default.
* Retrying an operation without an identity or duplicate policy.
* Assuming a timeout means the remote operation did not happen.
* Logging authorization headers or complete request bodies.
* Treating absent and `null` fields as interchangeable.
* Making a new client depend on a field an old server does not emit.
* Sharing a persistent client or tracing context across tenants in a worker.

## Senior Engineer Thinking

The important design question is not “which HTTP library should we use?” It is “what fact must cross the boundary, who owns it, and what can each party know after every failure?” A small typed port is valuable when it preserves those answers rather than merely renaming a generic client.

## Exercises

1. Specify a contract for a synchronous inventory reservation call, including request identity, deadline, response states, and duplicate behavior.
2. Map a JSON response with absent, `null`, and valued fields into a typed PHP result without losing the distinctions.
3. Design an asynchronous image-processing API with an accepted response, status lookup, expiration, and failure state.
4. List the data that may cross a payment-service boundary and the data that must remain internal.

## Review Questions

* Why does a successful connection not prove a business operation succeeded?
* Which boundary state should be durable for asynchronous work?
* Why is a transport response a poor domain type?
* What does a client timeout say, and what does it not say?
* Which changes are compatibility events?
* Which network-boundary data must never appear in ordinary logs?

## Summary

A network call crosses process, ownership, timing, and failure boundaries. Define typed messages, authentication, deadlines, duplicate behavior, compatibility, and observability explicitly. Map transport data at a narrow adapter, validate representations and limits, and design for the ambiguous result in which the caller times out after the receiver has completed the work.

## References

- [PHP manual: Streams](https://www.php.net/manual/en/book.stream.php)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OWASP: API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x00-header/)
- [Chapter 39 — Serialization](../../volumes/03-php-object-model/039-serialization.md)
- [Chapter 151 — Deserialization](../../volumes/10-security/151-deserialization.md)
