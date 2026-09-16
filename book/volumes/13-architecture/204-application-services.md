---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 204
title: Application Services
slug: application-services
status: complete
summary: ../../_ai/chapter-summaries/204-application-services-summary.md
---

# Chapter 204 — Application Services

An application service implements a use case at the boundary of the application. It coordinates authorization, repositories, domain objects, transactions, domain events, and external ports. It translates a transport request into a command and a domain result into a response model.

It should coordinate rather than become the domain's dumping ground. Rules about whether an order may be placed belong in the aggregate or domain service; the application service decides which participants are loaded, which transaction surrounds them, and what happens after commit.

## Why this matters

Without an application boundary, controllers duplicate transaction and authorization steps, while domain objects become aware of HTTP and queues. A focused use-case handler gives every entry point the same sequence:

```text
input → authenticate/authorize → load → domain decision → persist → commit → publish
```

The sequence is not automatically atomic. Mark which steps are inside the database transaction and which are asynchronous or retryable.

## Commands and results

Use typed commands at the boundary:

```php
<?php

declare(strict_types=1);

final readonly class PlaceOrder
{
    /** @param list<array{sku: string, quantity: int}> $lines */
    public function __construct(
        public int $customerId,
        public array $lines,
        public string $requestKey,
    ) {
    }
}

final readonly class OrderReceipt
{
    public function __construct(public string $orderId, public string $status)
    {
    }
}
```

The HTTP adapter can validate JSON and construct `PlaceOrder`; a queue consumer or CLI command can construct the same command from its own trusted boundary. The command must still be authorized in context.

## A use-case handler

```php
interface Transaction
{
    /** @template T */
    public function run(Closure $operation): mixed;
}

final class PlaceOrderHandler
{
    public function __construct(
        private CustomerRepository $customers,
        private OrderRepository $orders,
        private IdempotencyStore $keys,
        private OrderFactory $factory,
        private Transaction $transaction,
        private EventPublisher $events,
    ) {
    }

    public function handle(PlaceOrder $command, int $actorId): OrderReceipt
    {
        $existing = $this->keys->find($actorId, $command->requestKey);
        if ($existing !== null) {
            return $existing;
        }

        return $this->transaction->run(function () use ($command, $actorId): OrderReceipt {
            $customer = $this->customers->ownedBy($command->customerId, $actorId);
            if ($customer === null) {
                throw new DomainException('Customer unavailable');
            }

            $order = $this->factory->place($customer, $command->lines);
            $this->orders->save($order);
            $receipt = new OrderReceipt($order->id(), 'placed');
            $this->keys->remember($actorId, $command->requestKey, $receipt);
            $this->events->record($order->releaseEvents());

            return $receipt;
        });
    }
}
```

This example shows the shape, not a complete transaction implementation. The idempotency lookup and insert need a database uniqueness constraint to handle two concurrent requests. Publishing should use an outbox or a transaction-aware publisher so a committed order cannot lose its event.

## Authorization and transaction boundaries

Authenticate the actor before invoking the use case, but perform object-level authorization where the object is loaded. Do not trust `customerId` merely because it is present in a command. A queue consumer needs an explicit actor or service identity and a policy for delegated work.

Keep database writes and invariant checks in one transaction where required. Do not hold a transaction open during email, HTTP, or slow provider calls. Commit durable state, then enqueue an outbox event or use a documented workflow for the external effect. If a provider must participate in a business operation, model compensation and reconciliation rather than pretending a local transaction spans it.

## Error and response policy

Map domain failures to stable application outcomes at the boundary. Do not return SQL exceptions or provider payloads as API responses. Distinguish validation, forbidden access, conflict, unavailable dependency, and retryable failure so callers can make correct decisions.

A command handler should have an idempotency policy for retried requests. Store the result or a durable status with the request key, scope it to the actor or tenant, and define what happens when the same key is reused with different input.

## Testing and operations

Unit-test the handler's orchestration with focused fakes or mocks: wrong actor, missing object, domain rejection, duplicate request, and event recording. Integration-test the real transaction, uniqueness constraint, outbox, and authorization query. End-to-end tests should cover one representative transport path.

Instrument use-case name, outcome, duration, retry count, and correlation ID. Redact commands containing secrets or personal data. A slow handler metric should separate database time, external dependency time, and queue wait where possible.

## Exercises

1. Add an idempotency conflict when a request key is reused with a different payload.
2. Mark every line in a handler as inside or outside the transaction and design the outbox boundary.
3. Add a queue-consumer entry point that supplies a service identity and preserves authorization context.

## Review questions

- What belongs in an application service rather than a domain service?
- Why is authorization tied to the loaded object and actor context?
- Which steps must not occur while a database transaction is open?
- How does idempotency interact with concurrent requests?
- What should integration tests prove beyond a mocked handler test?

## Summary

Application services implement use cases by coordinating authorization, repositories, domain decisions, transactions, idempotency, and events. Keep domain rules in domain objects and services, make transaction and external-effect boundaries explicit, return stable outcomes, and test orchestration with real persistence at critical seams.

## References

- [Martin Fowler: Service Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [Martin Fowler: Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html)
- [PHP manual: Closures](https://www.php.net/manual/en/functions.anonymous.php)
