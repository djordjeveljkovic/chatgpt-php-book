---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 203
title: Domain Events
slug: domain-events
status: complete
summary: ../../_ai/chapter-summaries/203-domain-events-summary.md
---

# Chapter 203 — Domain Events

A domain event is an immutable statement that something meaningful happened in the domain: `OrderPlaced`, `PaymentAuthorized`, or `MembershipSuspended`. It is a fact in past tense, with a stable identity and the information a consumer needs to react.

Events can decouple policies without making the system magically consistent. The important design questions are when an event is created, when it becomes durable, who owns its schema, and what happens when delivery or consumption fails.

## Why this matters

A checkout use case may need to update an order, record payment, send a receipt, update search, and notify analytics. Calling every secondary concern from the order entity creates coupling. A domain event lets the order publish a fact while separate handlers react to it.

Define an event as data, not as a command to a named consumer:

```php
<?php

declare(strict_types=1);

interface DomainEvent
{
    public function eventId(): string;
    public function occurredAt(): DateTimeImmutable;
}

final readonly class OrderPlaced implements DomainEvent
{
    /** @param list<array{sku: string, quantity: int}> $lines */
    public function __construct(
        private string $id,
        private int $customerId,
        private array $lines,
        private DateTimeImmutable $at,
        private string $idValue,
    ) {
    }

    public function eventId(): string { return $this->idValue; }
    public function occurredAt(): DateTimeImmutable { return $this->at; }
    public function orderId(): string { return $this->id; }
    public function customerId(): int { return $this->customerId; }
    /** @return list<array{sku: string, quantity: int}> */
    public function lines(): array { return $this->lines; }
}
```

An event ID supports deduplication. The event should contain stable identifiers and facts, not an entire mutable ORM object or a secret token. Keep sensitive data out of messages unless its access and retention policy are explicit.

## Raising and collecting events

An aggregate can collect events as it changes state:

```php
final class Order
{
    /** @var list<DomainEvent> */
    private array $recorded = [];

    public function __construct(private string $id, private string $status = 'draft')
    {
    }

    public function place(int $customerId, array $lines, DateTimeImmutable $now): void
    {
        if ($this->status !== 'draft' || $lines === []) {
            throw new DomainException('Order cannot be placed');
        }

        $this->status = 'placed';
        $this->recorded[] = new OrderPlaced(
            $this->id,
            $customerId,
            $lines,
            $now,
            bin2hex(random_bytes(16)),
        );
    }

    /** @return list<DomainEvent> */
    public function releaseEvents(): array
    {
        $events = $this->recorded;
        $this->recorded = [];

        return $events;
    }
}
```

The aggregate should not publish to a network from inside `place()`. The application boundary decides how state and events become durable. If persistence fails, the event must not escape as a fact that did not commit.

## The outbox boundary

A common reliable pattern writes the aggregate state and an outbox row in one database transaction. A separate publisher reads pending rows, publishes them, and marks them delivered. This avoids the dual-write gap where the database commits but the message publish fails, or the message publishes before the transaction rolls back.

An outbox still delivers at least once. The publisher may crash after sending and before marking the row, so consumers need an event ID and an idempotency record. Keep the outbox payload versioned, bounded, and observable. Retain failed rows for investigation and define replay policy.

## Domain events and integration events

A domain event names a fact inside the domain model. An integration event is a versioned message published across a process or ownership boundary. They may share data, but do not expose internal classes as a public schema by accident. An event mapper can translate `OrderPlaced` into a public `order.v1.placed` message with an explicit contract.

Commands ask a receiver to do something; events state that something happened. A message named `SendReceipt` is a command, while `ReceiptRequested` is a fact or request depending on its ownership. Naming affects retry, authority, and expectations.

## Evolution and consumers

Add fields in a backward-compatible way where consumers tolerate them, or publish a new version. Do not rename or change the meaning of a field silently. Define enum evolution, unknown-field behavior, timestamp format, identifiers, and retention.

Consumers should validate message shape, authorize the action they take, and handle duplicates. A valid signature proves who published a message; it does not prove the business action is still authorized.

## Testing and operations

Unit-test aggregate transitions and recorded events. Test the outbox transaction with a real database, publisher retry with a fake broker, and consumer idempotency with duplicate messages. Contract-test integration event schemas between teams.

Observe event lag, publish failures, dead-letter counts, duplicate suppression, handler duration, and event version. Include correlation and causation IDs without putting secrets in the event. A replay tool must be permissioned and idempotency-aware.

## Exercises

1. Add an `OrderCancelled` event and define when it can be raised.
2. Design an outbox table with event ID, type, version, payload, attempts, and next-attempt time.
3. Evolve an event by adding a field and write the compatibility rule for an old consumer.

## Review questions

- What makes an event a fact rather than a command?
- Why should an aggregate avoid publishing directly to a broker?
- How does an outbox reduce the dual-write gap, and what failure remains?
- Why do consumers need idempotency when a publisher is reliable?
- How are domain and integration event schemas related but different?

## Summary

Use immutable domain events to express meaningful facts and decouple secondary policies. Collect events with aggregate changes, persist state and outbox records atomically, publish with at-least-once expectations, version public schemas, and make consumers idempotent and observable.

## References

- [Martin Fowler: Domain Event](https://martinfowler.com/eaaDev/DomainEvent.html)
- [Martin Fowler: Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Enterprise Integration Patterns: Event Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/EventMessage.html)
