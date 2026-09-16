---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 244
title: Queues
slug: queues
status: complete
summary: ../../_ai/chapter-summaries/244-queues-summary.md
---

# Chapter 244 — Queues

## Why This Matters

A queue decouples a producer from a consumer in time. It can absorb a burst, isolate a slow dependency, and let a request return before nonessential work finishes. It cannot create capacity or guarantee delivery by itself. The queue introduces age, ordering, visibility, retry, retention, and operational failure modes.

Choose a queue because the business operation can tolerate its delivery contract. A password-reset email may be pending for seconds; a fraud decision may need a synchronous answer; a cache refresh may be safely coalesced. “Put it on a queue” is an architecture decision, not a performance spell.

## Queue Model

```text
producer → durable queue → consumer → effect
    │            │             │
  accept       backlog       retry/ack
```

Important quantities are arrival rate, service rate, depth, oldest age, in-flight count, processing time, and failure rate. A queue is healthy when its age and outcomes meet the business contract, not merely when its storage accepts messages.

Little's Law gives a useful approximation for a stable system:

```text
items in system = arrival rate × average time in system
```

If arrivals exceed long-term service capacity, depth grows. A queue can make the overload less visible to the caller while making completion later. Tell the caller whether work is accepted, pending, completed, or rejected.

## Queue Types and Semantics

The product of a queue includes:

* durability and retention;
* acknowledgment and redelivery behavior;
* visibility or lease timeout;
* ordering and partitioning;
* maximum message and batch size;
* retry and failure transport;
* delay and scheduling support;
* deduplication or idempotency expectations;
* monitoring and replay controls.

A FIFO queue can preserve order within a partition while reducing parallelism. A work queue can maximize throughput while allowing any worker to receive any message. A topic or event stream can fan out one event to several consumers, each with its own offset and backlog.

## Commands and Events

A command asks an owner to perform work and should have an operation identity. An event records a fact that occurred and may be consumed by several independent readers. Mixing the two makes ownership and retry semantics unclear.

```text
command: ReserveInventory
event:   InventoryReserved
```

The command must be authorized and validated by its owner. The event should describe a committed fact and include enough version and identity metadata for consumers to process late or duplicate delivery.

## A Message Producer

Publishing should make acceptance visible:

~~~php
<?php

declare(strict_types=1);

final readonly class PublishResult
{
    public function __construct(
        public string $messageId,
        public bool $accepted,
    ) {
        if ($messageId === '') {
            throw new InvalidArgumentException('Missing message ID');
        }
    }
}

interface WorkQueue
{
    /** @param array<string, mixed> $payload */
    public function publish(string $type, string $operationId, array $payload): PublishResult;
}

function enqueueEmail(WorkQueue $queue, string $operationId, int $userId): PublishResult
{
    if ($operationId === '' || $userId <= 0) {
        throw new InvalidArgumentException('Invalid email operation');
    }

    return $queue->publish('SendWelcomeEmail', $operationId, ['user_id' => $userId]);
}
~~~

The producer should not report “email sent” when it has only published a command. It should report accepted or pending and let the owner expose completion. The queue adapter should apply size, schema, authorization, and durability rules.

## Consumers and Concurrency

A consumer claims work, processes it, and acknowledges according to the delivery contract. Limit concurrency to what the database, provider, and worker host can safely handle. A large consumer fleet can overwhelm a small downstream pool.

Use separate queues or consumer pools for work with different latency, priority, or resource profiles. A bulk export should not starve account recovery. A slow class of messages can need a longer lease and a different worker memory limit.

Partition by a key when operations for one aggregate must be ordered. Measure hot partitions and avoid a global lock around all messages. If a message is independent, do not impose ordering that prevents safe parallelism.

## Batching

Batching reduces per-message connection, serialization, and transaction overhead. It increases memory, lock duration, failure scope, and replay complexity. Define whether a batch is atomic, independently committed, or partially successful.

An acknowledgment for a batch should not hide which items failed. Store per-item identity and outcome when a replay must avoid repeating successful effects. Bound batch count, bytes, and age so a quiet queue does not wait forever for a full batch.

## Backlog and Priority

Depth is useful but incomplete. Track oldest-message age and end-to-end time. A queue with 10,000 tiny messages may be healthier than one with 100 large messages if its age and dependency load are lower.

Priority can be implemented with separate queues, partitions, or broker features. Separate queues make worker capacity explicit but can leave one class idle while another is overloaded. A single priority queue can create starvation. Define fairness and aging rules.

When backlog grows, admit less work, reduce optional work, coalesce superseded commands, increase safe capacity, or communicate delay. Do not silently discard important commands. Dropping stale work requires an explicit domain policy and audit trail.

## PHP Worker Lifecycle

PHP queue workers are often long-running processes. Reset tenant, locale, authorization, transaction, tracing, and memory-heavy state after every message. Use a graceful shutdown: stop claiming new messages, finish or release the current one within a deadline, flush telemetry, and exit. Recycle workers after a bounded lifetime when that is part of the memory policy, while investigating retention.

The queue's lease must exceed normal processing time with margin, or be renewed according to the transport contract. A forced process kill can cause redelivery; idempotent handlers make that safe. See [Chapter 243 — Message Delivery](./243-message-delivery.md) and [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md).

## Testing

Use a real broker, compatible emulator, or contract-tested adapter for acknowledgment, lease, ordering, delay, and redelivery behavior. Test duplicate messages, out-of-order delivery, full queue, unavailable broker, expired lease, graceful shutdown, partial batches, poison messages, and downstream saturation.

Assert the business state and queue state together. A test that sees an acknowledgment but does not inspect the durable effect can pass while the handler loses work.

## Security

Messages can contain sensitive data and can be replayed. Encrypt or protect transport and storage as required, validate producer identity, authorize commands, scope tenant data, restrict management and replay operations, and redact payloads from ordinary logs. A queue name is not an authorization boundary.

## Common Mistakes

* Using a queue for work whose caller needs a synchronous result.
* Reporting completion when only publication succeeded.
* Scaling consumers without checking downstream capacity.
* Assuming FIFO ordering across all workers and partitions.
* Acknowledging a batch without recording item-level outcomes.
* Measuring depth while ignoring age and end-to-end latency.
* Sharing request state between long-running PHP messages.
* Giving replay or queue-management access to ordinary application users.

## Senior Engineer Thinking

Ask what temporal decoupling buys, what delay the user accepts, who owns the result, and which failures the queue can preserve. Then specify delivery, ordering, retry, retention, capacity, and replay as one contract. A queue is useful when those semantics are clearer after introducing it.

## Exercises

1. Choose queue or synchronous handling for welcome email, payment authorization, search indexing, and cache refresh. Explain each contract.
2. Design three consumer pools for interactive, bulk, and provider-limited work. State the concurrency budget for each.
3. Define a batch format with partial success and safe replay.
4. Measure queue depth, oldest age, service rate, and end-to-end completion for a controlled burst.

## Review Questions

* What can a queue absorb, and what can it not create?
* Why must accepted work be distinguished from completed work?
* How do commands differ from events?
* What does batching trade for throughput?
* Why are age and end-to-end latency needed in addition to depth?
* Which PHP state must be reset between messages?

## Summary

A queue decouples production from consumption while introducing age, ordering, leases, retries, retention, and replay semantics. Define the delivery contract, distinguish commands from events, bound consumer concurrency by downstream capacity, partition only when useful, make batches and priority explicit, keep PHP workers clean, and measure customer-visible completion rather than queue depth alone.

## References

- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Enterprise Integration Patterns: Message Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageChannel.html)
- [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md)
- [Chapter 243 — Message Delivery](./243-message-delivery.md)
