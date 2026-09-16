---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 195
title: Hexagonal Architecture
slug: hexagonal-architecture
status: complete
summary: ../../_ai/chapter-summaries/195-hexagonal-architecture-summary.md
---

# Chapter 195 — Hexagonal Architecture

## Why This Matters

Hexagonal architecture, also called Ports and Adapters, keeps application and domain behavior independent from external technology. The core defines the ports it needs; adapters connect those ports to HTTP, CLI, databases, queues, files, and providers.

The hexagon is a visual metaphor, not a required package layout. Its test is dependency direction: the core does not import a framework, database driver, or provider SDK to express a business rule.

## Inside and Outside

There are two useful port directions:

* **Driving or primary ports:** operations outside actors invoke, such as PlaceOrder or GenerateInvoice.
* **Driven or secondary ports:** capabilities the core needs, such as OrderRepository, Clock, PaymentGateway, or EventPublisher.

Adapters implement or call these ports. An HTTP controller is a driving adapter; a PDO repository and payment client are driven adapters. The composition root connects them.

~~~php
<?php

declare(strict_types=1);

final readonly class PlaceOrder
{
    public function __construct(
        public int $customerId,
        /** @var list<OrderItem> */
        public array $items,
    ) {
    }
}

interface PlaceOrderPort
{
    public function execute(PlaceOrder $command): int;
}

interface OrderStore
{
    public function save(PlaceOrder $command): int;
}

final class PlaceOrderService implements PlaceOrderPort
{
    public function __construct(private OrderStore $orders)
    {
    }

    public function execute(PlaceOrder $command): int
    {
        if ($command->items === []) {
            throw new InvalidArgumentException('Order needs an item');
        }

        return $this->orders->save($command);
    }
}
~~~

PHP enforces array at runtime; the PHPDoc list<OrderItem> shape can be checked by static analysis. A dedicated OrderItem type makes the port clearer when item rules become significant.

Use PHP runtime types for the enforced boundary and static analysis for richer collection shapes.

## Adapters Translate Contracts

An adapter maps external data and errors to the port's vocabulary. A PDO adapter owns SQL, parameter binding, transaction details, and database-specific exceptions. A provider adapter owns authentication, timeouts, response parsing, and retry classification. The core should receive a domain result or a stable failure, not a vendor response array.

Keep adapters thin but not mindless. Security checks, input limits, idempotency keys, and correlation IDs must be preserved across the translation. Never let an adapter silently turn a timeout or signature failure into success.

## Composition Root and Configuration

Construct the core and adapters at the application edge. The composition root can select a real or in-memory adapter for a context, inject typed configuration, and define lifetimes. The domain does not ask a container for a service.

A port should be as narrow as the use case requires. An OrderRepository with twelve methods may be a persistence abstraction rather than a useful application port. Split read and write capabilities or define a focused query when different callers need different guarantees.

## Testing the Hexagon

The core can be tested through driving ports with in-memory driven adapters or fakes. These tests run quickly and verify domain behavior. Each driven adapter gets integration tests against its real boundary. Driving adapters get feature or protocol tests for routing, serialization, authentication, and content negotiation.

Contract tests ensure adapters implement the port semantics. For a payment gateway, test amount and currency mapping, timeout classification, idempotency propagation, and provider errors. A fake that returns a successful charge for every request cannot prove those translations.

## Transactions and Distributed Work

A transaction boundary belongs where the use case knows the required consistency. A driven transaction port can provide a unit of work, or the application service can coordinate repositories through an explicit transaction manager. Do not let the hexagonal boundary imply that a database transaction includes a remote provider.

For a database write and event publication, use an outbox adapter or durable event port. For a provider call that may commit before timing out, use a provider idempotency key and reconciliation. Architecture separates dependencies; it does not remove distributed-systems failure.

## Failure and Threat Analysis

* **Framework leakage:** domain imports Request or ORM models. Map to commands and value objects.
* **Anemic port:** every infrastructure detail appears in the interface. Define a use-case capability.
* **Fake mismatch:** tests pass with an unrealistic fake. Contract-test adapters and fakes.
* **Composition drift:** production and test graphs differ. Keep composition explicit and verify configuration.
* **Hidden I/O:** a port call performs slow or remote work without a contract. Document timeout and failure.
* **Authorization gap:** an adapter trusts caller IDs. Pass authenticated context and authorize the resource.
* **Transaction illusion:** a port spans database and provider without recovery. Use outbox, workflow, or compensation.

## Exercises

1. Draw primary and secondary ports for a refund use case and identify one adapter for each.
2. Replace a provider SDK type in a domain service with a focused port and stable failure taxonomy.
3. Write a contract test shared by a PDO repository and an in-memory repository.
4. Design a hexagonal outbox adapter and explain how it recovers after a publish failure.

## Review Questions

1. What is the difference between a driving and driven port?
2. Which code belongs in an adapter?
3. Why should a port be narrower than a repository's entire API?
4. What can a fake prove, and what still needs integration testing?
5. Why does hexagonal architecture not create a distributed transaction?
6. Where should composition and configuration live?

## Summary

Hexagonal architecture keeps core behavior independent from external technology through driving and driven ports. Adapters translate data and failures, composition roots connect the graph, and tests exercise the core, adapters, and protocols at their true boundaries. Ports clarify dependencies but do not remove authorization, transaction, timeout, idempotency, or recovery design.

## References

- [Alistair Cockburn: Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [PHP Manual: Interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)
- [PHP-FIG PSR-11: Container Interface](https://www.php-fig.org/psr/psr-11/)
- [Martin Fowler: Inversion of Control](https://martinfowler.com/bliki/InversionOfControl.html)

