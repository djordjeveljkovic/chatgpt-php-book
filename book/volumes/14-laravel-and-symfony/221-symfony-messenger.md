---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 221
title: Symfony Messenger
slug: symfony-messenger
status: complete
summary: ../../_ai/chapter-summaries/221-symfony-messenger-summary.md
---

# Chapter 221 — Symfony Messenger

## Why This Matters

Symfony Messenger gives an application a message bus for dispatching commands, events, and queries, either immediately or through a transport. A message can be handled in the current process or serialized, queued, retried, and delivered by a worker.

The bus API is convenient, but a transport is a distributed boundary. A worker can receive a message more than once, run it after a deploy, fail after a side effect, or process it after authorization has changed. Reliable Messenger code starts with a durable message contract and an explicit failure policy.

## Messages and Handlers

A message should carry small, stable data: identifiers, operation IDs, and the schema version needed to interpret it. Avoid serializing ORM graphs, request objects, secrets, or a stale authorization decision.

~~~php
<?php

declare(strict_types=1);

namespace App\Message;

final readonly class GenerateInvoicePdf
{
    public function __construct(
        public int $invoiceId,
        public int $tenantId,
        public string $operationId,
        public int $schemaVersion = 1,
    ) {
    }
}
~~~

A handler performs the operation:

~~~php
<?php

declare(strict_types=1);

namespace App\MessageHandler;

use App\Message\GenerateInvoicePdf;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class GenerateInvoicePdfHandler
{
    public function __construct(private InvoicePdfGenerator $generator)
    {
    }

    public function __invoke(GenerateInvoicePdf $message): void
    {
        $this->generator->generate(
            invoiceId: $message->invoiceId,
            tenantId: $message->tenantId,
            operationId: $message->operationId,
        );
    }
}
~~~

Attribute-based handler registration and configuration details vary by Symfony version; the concept is stable. The handler should load current state with tenant scope, authorize where required, and use the operation ID to make effects idempotent.

## Buses and Routing

A message bus dispatches an Envelope containing a message and optional stamps. Routing configuration can send selected messages to a transport; messages without an asynchronous route may be handled synchronously. Keep command, event, and query buses separate when their middleware and failure policies differ, or use one bus when a small application benefits from simpler wiring.

~~~php
<?php

declare(strict_types=1);

use App\Message\GenerateInvoicePdf;
use Symfony\Component\Messenger\MessageBusInterface;

final class InvoiceApplicationService
{
    public function __construct(private MessageBusInterface $bus)
    {
    }

    public function requestPdf(int $invoiceId, int $tenantId, string $operationId): void
    {
        $this->bus->dispatch(new GenerateInvoicePdf(
            invoiceId: $invoiceId,
            tenantId: $tenantId,
            operationId: $operationId,
        ));
    }
}
~~~

Dispatch is not proof that a message was durably accepted. The transport, serializer, broker acknowledgment, and application transaction determine the guarantee. Do not report a durable state transition before the required write and message publication are coordinated.

## Transactions and Dispatch Timing

If a message refers to data created in a database transaction, dispatch it after the transaction commits. Dispatching inside the transaction can let a fast worker query data that later rolls back. Messenger provides middleware and configuration options for dispatch-after-current-bus behavior in supported versions; verify the exact setup for the installed release.

When a database write and enqueue must be coordinated, use an outbox or a transaction-aware transport strategy. An application-level dispatch call cannot make an arbitrary database and broker transaction atomic. The outbox record should contain a durable event ID, payload version, and delivery state.

## Delivery, Retry, and Failure Transports

A transport commonly provides at-least-once delivery. A worker may crash after a provider commits and before the broker acknowledges. Make handlers idempotent with a unique operation ID, a durable effect record, or a provider idempotency key.

Classify failures:

* transient network, lock, or capacity errors may retry;
* invalid input and permanent missing data should be rejected or sent to a failure transport;
* provider timeouts need reconciliation before retrying an external effect;
* authorization revocation may require dropping or quarantining the message;
* poison payloads must not consume all worker capacity.

Configure bounded retries, backoff, and a failure transport for inspection and replay. Replaying a failed message should preserve its operation ID, validate its schema, and re-check authorization and current state. Never replay blindly after a security or business policy change.

## Middleware and Stamps

Messenger middleware can add transaction handling, validation, logging, routing, dispatch-after-current-bus behavior, and security context. Stamps carry metadata on an Envelope, such as a bus name or transport settings. Keep middleware ordering explicit: validation and authorization must happen before side effects, while acknowledgment and transaction behavior must match the transport guarantee.

Do not trust a user ID or tenant ID in a message as sufficient authorization. It identifies the intended context; the handler must verify current membership or use an authenticated service identity. Redact message bodies and stamps from logs when they contain personal data or credentials.

## Workers and Concurrency

A Messenger worker is a long-running PHP process. Configure time and memory limits, graceful signals, queue selection, prefetch or visibility behavior, and restart strategy according to the transport. Reset request-scoped and tenant-scoped state between messages. A service that is safe for one short request can leak mutable state across thousands of messages.

Two workers can handle messages for the same aggregate concurrently. Use optimistic versions, database locks, per-key serialization, or a state machine where ordering matters. A transport's ordering is not a domain invariant unless it is explicitly configured and tested.

## Serialization and Compatibility

Message serialization is a public contract between producer and consumer. Prefer stable scalar fields and explicit schema versions. Add fields compatibly, retain old readers during deployment, and handle unknown or missing fields according to policy. Avoid serializing PHP object internals or framework model objects whose shape changes across versions.

Validate message size, nesting, types, and allowed classes at the consumer boundary. Do not enable unsafe object deserialization for untrusted payloads. Sign or authenticate messages when the transport threat model requires it, and protect broker credentials.

## Observability and Security

Log message type, operation ID, tenant, attempt, transport, duration, and outcome using redaction. Propagate correlation and trace IDs through envelopes and downstream calls. Measure queue depth, oldest message age, throughput, retries, failure-transport size, worker memory, and handler latency.

Least-privilege transport credentials should allow a worker only the queues and operations it needs. A private broker does not remove message validation or authorization. A failed policy lookup must fail closed or produce a bounded retryable error, not grant access.

## Failure and Threat Analysis

* **Duplicate handling:** a crash before acknowledgment repeats the message. Make effects idempotent.
* **Premature dispatch:** a message runs before a transaction commits. Use after-commit dispatch or an outbox.
* **Poison message:** invalid input retries forever. Classify and send it to a failure transport.
* **Stale authorization:** a queued command outlives membership. Re-check policy at handling time.
* **Schema drift:** a new producer breaks an old worker. Version payloads and deploy compatibly.
* **Worker leakage:** state or memory persists across messages. Reset context and use bounded restarts.
* **Ordering race:** two messages update one aggregate concurrently. Use versions, locks, or serialization.
* **Secret exposure:** logs or serialized messages contain credentials. Keep payloads minimal and redact diagnostics.

## Testing Messenger Applications

Unit-test message validation, handler decisions, idempotency, and failure classification. Integration-test serializers, routing, transport acknowledgment, failure transport, database transactions, and provider adapters. Feature-test the application operation that dispatches the message and its public response.

Test duplicate delivery, retries, backoff classification, malformed and old-version messages, authorization revocation, process termination, and out-of-order messages. Use a real broker or transport emulator for guarantees that an in-memory transport cannot reproduce.

## Exercises

1. Design a GenerateInvoicePdf message with tenant, operation, schema version, retry, and authorization policy.
2. Configure a message that dispatches after a database commit and compare it with an outbox design.
3. Write a handler test for duplicate delivery, stale authorization, provider timeout, and permanent invalid input.
4. Design a failure transport replay procedure that preserves operation identity and prevents unsafe replays.
5. Measure queue age and worker memory while processing a large import.

## Review Questions

1. What changes when a Messenger message is routed to a transport?
2. Why should messages carry identifiers rather than ORM object graphs?
3. What crash window requires idempotent handlers?
4. How can dispatch before commit create a missing-data race?
5. Why must a failed message be classified before retry?
6. Which metadata should cross a message boundary?
7. Why does an in-memory transport provide weaker evidence than a real broker?

## Summary

Symfony Messenger dispatches commands and events synchronously or through transports, but queued handling has at-least-once, serialization, retry, ordering, and worker-lifecycle semantics. Keep messages small and versioned, coordinate dispatch with transactions or an outbox, make handlers idempotent, bound retries, re-check authorization, reset worker state, and observe queue age and failure transports.

## References

- [Symfony Messenger](https://symfony.com/doc/current/messenger.html)
- [Symfony Messenger: Transports](https://symfony.com/doc/current/messenger.html#transports)
- [Symfony Messenger: Middleware](https://symfony.com/doc/current/messenger.html#middleware)
- [Symfony Messenger: Testing](https://symfony.com/doc/current/messenger.html#testing)
- [Symfony Serializer](https://symfony.com/doc/current/serializer.html)
- [microservices.io: Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html)

