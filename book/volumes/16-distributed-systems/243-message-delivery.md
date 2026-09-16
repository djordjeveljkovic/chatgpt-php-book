---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 243
title: Message Delivery
slug: message-delivery
status: complete
summary: ../../_ai/chapter-summaries/243-message-delivery-summary.md
---

# Chapter 243 — Message Delivery

## Why This Matters

A message broker stores and transfers work, but it does not make delivery exactly once by default. A producer can publish and crash before recording success. A consumer can perform the side effect and crash before acknowledgment. The broker can redeliver after a lease expires.

Choose a delivery contract deliberately: at-most-once, at-least-once, or an effectively-once business effect built from durable identity and idempotent handling. The words describe observable behavior, not a guarantee that hardware and networks never fail.

## Delivery Models

**At-most-once** acknowledges or removes a message before processing. It avoids duplicates at the cost of losing work when the consumer fails.

**At-least-once** acknowledges after processing. It preserves work across many consumer failures but permits duplicates in the crash window.

**Effectively once** makes repeated deliveries produce one business outcome through idempotency, unique constraints, version checks, or an atomic local state transition. It still requires clear semantics for external effects and unknown completion.

```text
receive → effect → acknowledge
            ↑
      crash window
```

There is no universal best choice. Losing a cache refresh may be acceptable; losing a payment command is not. A payment handler may use at-least-once transport and effectively-once business effect.

## Publish and Consume Boundaries

Publishing has its own failure window:

```text
database commit → process crashes → message never published
```

The transactional outbox pattern records the event in the same database transaction as the business change. A separate publisher reads the outbox, publishes with an event identity, and marks the row as sent. The publisher may repeat a message, so consumers remain idempotent.

Consuming has the inverse window. A handler should commit local state and record the message or operation identity before acknowledging. If the broker and database cannot participate in one transaction, use an inbox or deduplication record at the consumer boundary.

## Envelope and Identity

Keep transport metadata separate from the business payload:

```text
message_id       delivery identity
operation_id     intended business effect
event_type       stable semantic name
schema_version   decoding contract
occurred_at      producer event time
trace_context    correlation only
payload          validated domain data
```

An event ID may be unique per event while a command ID is reused on retries. Preserve both when the distinction matters. Do not use a timestamp as a unique ID; clocks can collide or move.

## A Typed Envelope

The envelope makes validation and routing visible:

~~~php
<?php

declare(strict_types=1);

final readonly class MessageEnvelope
{
    /** @param array<string, mixed> $payload */
    public function __construct(
        public string $messageId,
        public string $operationId,
        public string $type,
        public int $schemaVersion,
        public array $payload,
    ) {
        if ($messageId === '' || $operationId === '' || $type === '' || $schemaVersion < 1) {
            throw new InvalidArgumentException('Invalid message envelope');
        }
    }
}

interface MessageHandler
{
    public function handle(MessageEnvelope $message): void;
}

function dispatch(MessageEnvelope $message, MessageHandler $handler): void
{
    $handler->handle($message);
}
~~~

The adapter should reject unknown event types or unsupported schema versions according to a compatibility policy. It should not pass arbitrary decoded data into a handler that assumes a shape.

## Acknowledgment and Leases

Many brokers use an acknowledgment or visibility lease. The consumer must finish before the lease expires or renew it according to documented rules. Renewal does not make a handler idempotent: a process can still die after the side effect and before acknowledgment.

Acknowledging too early risks loss. Acknowledging too late increases redelivery and in-flight occupancy. Batch acknowledgment improves throughput but increases the replay scope after a failure; define how partial batch success is recorded.

Do not acknowledge a malformed or unauthorized message as successful if an operator needs to investigate it. Move it to a failure path with bounded retention. Do not endlessly redeliver a poison message and call the resulting load “reliability.”

## Ordering and Partitioning

Global ordering is expensive and often unnecessary. Partition by an aggregate or customer key when commands for that key must be serialized. This preserves local order while allowing independent keys to progress.

Partitioning creates hot keys. One customer with a large workload can dominate one partition while others are idle. Measure per-partition age and throughput. If order is not required, remove the constraint; if it is required, design a fair or coalescing policy.

Events can arrive late or out of order. Consumers should use event versions, sequence numbers, or commutative updates where possible. A consumer that blindly applies an older snapshot can regress state.

## Schema Evolution

Messages often outlive the producer deployment that created them. Add fields in a backward-compatible way, keep consumers tolerant of fields they do not need, and retain old decoding for the queue's maximum age. Version a semantic change rather than changing the meaning of a field in place.

A schema version is not validation by itself. Validate types, ranges, tenant scope, and authorization context before processing. Reject or quarantine unsupported versions with enough metadata for a safe replay after deployment.

## What PHP Does

A PHP queue worker is a long-running process in many deployments. Autoloaded classes and static state remain in memory between messages, while a message's tenant, locale, transaction, logger context, and trace context should not. Reset request-scoped state in a `finally` block and bound worker lifetime. See [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md).

Serialization and acknowledgment are separate operations. A successful `json_decode()` does not mean the payload is valid, and a successful `ack()` does not prove the business effect is durable.

## Testing

Test each delivery model with a fake broker that can crash the consumer at controlled points. Cover duplicate delivery, lost publish, delayed acknowledgment, expired lease, out-of-order messages, unsupported schema, malformed payload, partial batch success, and graceful shutdown.

Run a contract test for the envelope and a database integration test for outbox/inbox uniqueness. Assert business effects and durable records, not only acknowledgment calls.

## Security

Authenticate producers and consumers where the transport requires it, authorize message actions, validate tenant identity from trusted context, and protect replay tools. A message that contains `tenant_id` is not automatically authorized for that tenant. Redact secrets and personal data from envelopes, logs, and failure stores.

## Common Mistakes

* Claiming exactly-once delivery from a broker setting.
* Acknowledging before the business effect is durable.
* Assuming an outbox publisher never sends a duplicate.
* Using message ID where a domain operation ID is required.
* Letting leases expire during normal processing.
* Applying events without version or ordering protection.
* Making a breaking payload change while old messages remain queued.
* Reusing tenant or tracing state between PHP worker messages.

## Senior Engineer Thinking

Ask where the crash windows are, which facts are durable, and what repeated delivery does. A useful delivery contract says what may be lost, what may repeat, how order is scoped, how old messages decode, and how operators replay failures without bypassing authorization.

## Exercises

1. Compare at-most-once and at-least-once delivery for password-reset email, payment capture, and cache invalidation.
2. Design an outbox and inbox record with identities, state, retry count, retention, and tenant scope.
3. Simulate a worker crash after a side effect and before acknowledgment. Prove that the business effect remains singular.
4. Design a schema-evolution plan for a message that may remain queued for 24 hours.

## Review Questions

* Which crash window creates duplicate delivery?
* What does “effectively once” require beyond broker behavior?
* Why are event and operation identities different?
* What does a queue lease protect, and what does it not protect?
* Why can global ordering reduce throughput?
* Which state must be reset in a long-running PHP consumer?

## Summary

Message delivery is a contract about loss, duplication, ordering, acknowledgment, and schema lifetime. Use at-least-once transport with idempotent effects when preserving work matters, use outbox/inbox records for publish and consume gaps, keep event and operation identity distinct, bound leases and retries, evolve schemas compatibly, and test crash windows explicitly.

## References

- [Enterprise Integration Patterns: Idempotent Receiver](https://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html)
- [Enterprise Integration Patterns: Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md)
- [Chapter 242 — Idempotency](./242-idempotency.md)
