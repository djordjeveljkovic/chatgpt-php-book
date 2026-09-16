---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 205
title: Event-Driven Architecture
slug: event-driven-architecture
status: complete
summary: ../../_ai/chapter-summaries/205-event-driven-architecture-summary.md
---

# Chapter 205 — Event-Driven Architecture

Event-driven architecture (EDA) uses events to communicate that something happened. Producers publish facts; consumers subscribe and react independently. The style can reduce direct coupling and enable asynchronous work, but it introduces delivery, ordering, duplication, schema, and observability concerns.

An event is not merely a queue payload. The architecture needs explicit ownership of topics, schemas, consumer groups, retention, retries, and failure handling.

## Why this matters

A synchronous checkout that waits for email, search indexing, analytics, and fulfillment has high latency and a large failure surface. Publishing `OrderPlaced` allows those consumers to proceed independently. The order transaction still needs a durable handoff, and the customer-facing response must say whether secondary work is pending.

Choose the communication style deliberately:

- **Point-to-point command:** one owner is asked to act.
- **Publish/subscribe event:** multiple consumers may react to a fact.
- **Stream:** ordered records and offsets are part of the contract.
- **Workflow orchestration:** a coordinator tracks steps and compensation.

A broker does not choose semantics for you. “Asynchronous” does not mean “eventually correct” unless consumers, retries, and reconciliation are designed.

## Delivery guarantees

Most practical brokers provide at-least-once delivery or can redeliver after a consumer crash. Consumers must be idempotent. Exactly-once claims often describe a narrow broker operation, not the complete business effect in a database and an external provider.

A consumer can record an event ID before applying its effect only if both records share a transaction or the effect is otherwise idempotent:

```php
<?php

declare(strict_types=1);

interface ProcessedEventStore
{
    public function alreadyProcessed(string $consumer, string $eventId): bool;
    public function markProcessed(string $consumer, string $eventId): void;
}

final class OrderPlacedConsumer
{
    public function __construct(
        private ProcessedEventStore $processed,
        private Fulfillment $fulfillment,
    ) {
    }

    /** @param array<string, mixed> $event */
    public function handle(array $event): void
    {
        $id = $event['event_id'] ?? null;
        if (!is_string($id) || $id === '') {
            throw new InvalidArgumentException('Event ID is required');
        }
        if ($this->processed->alreadyProcessed(self::class, $id)) {
            return;
        }

        $this->fulfillment->reserve($event['order_id'], $id);
        $this->processed->markProcessed(self::class, $id);
    }
}
```

In production, `reserve` and `markProcessed` need a transaction or an idempotency key that closes the crash window. Validate schema and authorization before using event fields. A message from an internal broker is still input at the consumer boundary.

## Outbox and inbox patterns

The transactional outbox connects a local database commit to publication. The inbox or processed-event table makes consumer handling repeatable. Both require retention, cleanup, monitoring, and a policy for poison messages.

A publisher should use bounded attempts and backoff, preserve the original event ID, and move permanently failing records to a reviewable dead-letter state. A consumer should not acknowledge a message before its required durable effect succeeds. If an effect is intentionally best effort, record that policy and expose its failure separately.

## Choreography and orchestration

In choreography, each consumer reacts to events and may emit new events. It is simple for independent reactions but can make a business workflow difficult to see and can create event chains or cycles. In orchestration, a coordinator sends commands and tracks state. It centralizes workflow policy but becomes a critical component.

Use choreography for loosely coupled reactions such as analytics or search indexing. Use orchestration when ordering, compensation, deadlines, and human decisions are part of one business process. Draw the state machine before choosing.

## Schema, ordering, and evolution

Give every message an event ID, type, version, occurred-at time, producer, correlation ID, and payload. Define field types, required fields, unknown-field behavior, and privacy retention. Use stable identifiers rather than embedding mutable object graphs.

Ordering is usually scoped to a key such as an order ID, not global. Consumers must handle late and duplicate events. If two events can arrive in either order, model a state transition that accepts both or persist enough version information to reject stale updates.

Additive changes are often safer than changing meaning. Publish a new version when a field's semantics or required behavior changes. Consumer contract tests and a schema registry or reviewed schema repository can make compatibility visible.

## Testing and operations

Test producer transaction/outbox behavior, publisher retries, schema validation, duplicate delivery, out-of-order events, poison messages, dead letters, and consumer recovery. Use a real broker or compatible test environment for offset, visibility, and acknowledgment behavior; an in-memory array cannot prove those semantics.

Observe publish lag, consumer lag, retry counts, dead-letter volume, duplicate suppression, per-event latency, and handler failure categories. Include trace context without propagating secrets. A replay tool needs authorization, rate limits, and idempotency awareness.

## Exercises

1. Design an `OrderPlaced` topic contract with version, key, retention, and compatibility rules.
2. Add a crash-window test around an inbox record and a fulfillment effect. Choose an idempotency strategy.
3. Compare choreography and orchestration for a refund workflow with payment, inventory, and customer notification.

## Review questions

- Why does at-least-once delivery require idempotent consumers?
- What gaps do outbox and inbox patterns address?
- When is orchestration clearer than choreography?
- What does an event schema need beyond a payload array?
- Which broker semantics cannot be proven by an in-memory fake?

## Summary

Event-driven architecture separates producers and consumers through explicit facts, but it transfers complexity into delivery, idempotency, schema evolution, ordering, and operations. Use outbox and inbox boundaries, choose choreography or orchestration deliberately, and test and observe the real broker semantics.

## References

- [Martin Fowler: What do you mean by Event-Driven?](https://martinfowler.com/articles/201701-event-driven.html)
- [microservices.io: Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [PHP manual: JSON](https://www.php.net/manual/en/book.json.php)
