---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 177
title: Coupling
slug: coupling
status: complete
summary: ../../_ai/chapter-summaries/177-coupling-summary.md
---

# Chapter 177 — Coupling

## Why This Matters

Coupling is the amount of knowledge or coordination one part of a system needs from another. Some coupling is necessary: an order service must know the command it accepts and the result it promises. Accidental coupling makes every change expensive because a small modification requires unrelated modules, tests, deployments, or teams to move together.

The goal is not zero coupling. The goal is coupling that follows stable boundaries, is visible in interfaces, and is cheap to change when requirements change.

## Kinds of Coupling

A module can be coupled through:

* concrete classes, inheritance, or static calls;
* shared mutable state, globals, environment variables, or database tables;
* data shape, such as an array with undocumented keys;
* control flow, where one component must call another in a particular sequence;
* timing, ordering, or transaction boundaries;
* deployment, so two modules must be released together;
* failure behavior, such as a dependency that can throw, time out, or return partial data.

A direct call is not automatically bad. A stable standard-library call may be cheaper than an abstraction. Coupling becomes a design problem when the dependency is volatile, difficult to test, crosses a trust boundary, or leaks details that the caller should not know.

## Dependency Direction

Keep policy and domain decisions independent from volatile mechanisms such as HTTP clients, SQL drivers, framework containers, and vendor SDKs. Define a narrow port around the behavior the application needs and put the adapter at the edge:

~~~php
<?php

declare(strict_types=1);

interface PaymentGateway
{
    public function charge(int $amountCents, string $currency, string $reference): ChargeResult;
}

final class CheckoutService
{
    public function __construct(private PaymentGateway $payments)
    {
    }

    public function checkout(int $amountCents, string $currency, string $orderId): ChargeResult
    {
        if ($amountCents <= 0) {
            throw new InvalidArgumentException('Amount must be positive');
        }

        return $this->payments->charge($amountCents, $currency, $orderId);
    }
}
~~~

The service is coupled to the PaymentGateway contract, which is stable and meaningful to the domain. The Stripe or bank adapter can change without changing the checkout rule. Do not create an interface merely to mock a class; create a boundary where an independent variation or failure policy exists.

## Composition and Data Boundaries

Construct dependencies in a composition root rather than letting domain code discover them through a service locator or global container. Explicit construction makes ownership, lifecycle, and test substitutes visible.

Pass a typed command or value object across a boundary instead of an associative array whose keys are known only by convention. Keep transport DTOs, domain objects, and persistence records separate when their change rates differ. An adapter should translate errors and data shapes rather than making every caller learn a provider's response format.

For module or service boundaries, prefer stable IDs and explicit messages. Do not share internal database tables as an accidental API. Shared tables couple migrations, permissions, transaction assumptions, and deploy timing; if sharing is required, document the schema contract and ownership.

## Temporal and Failure Coupling

A caller that must invoke A, then B, then C in one request has temporal coupling. It may be correct for a transaction, but the sequence must be explicit and failure recovery must be designed. If an email can be delayed, write an outbox event and deliver it asynchronously rather than making checkout depend on SMTP availability.

Timeouts, retries, idempotency, and compensation are part of a dependency contract. A dependency that can return a timeout after committing is more strongly coupled than a pure function. Put this knowledge in an adapter or workflow service so domain code receives a small, stable failure taxonomy.

## Reducing Coupling Safely

Before extracting an abstraction, identify the change you are trying to isolate. Introduce a seam, characterize current behavior with tests, move one dependency behind it, and measure whether callers actually became simpler. An abstraction that exposes every detail of the concrete dependency merely moves coupling behind a new name.

Events reduce direct call coupling but introduce schema, delivery, ordering, duplication, and observability responsibilities. Use them when independent timing or deployment is valuable, not as a universal replacement for a clear function call.

## Failure and Threat Analysis

* **Global state:** tests and requests influence each other. Inject configuration, clocks, and stores.
* **Leaky interface:** callers depend on vendor fields. Map to a domain contract.
* **Shared database:** independent modules change each other's schema. Assign ownership or document a versioned contract.
* **Temporal failure:** a downstream timeout leaves partial work. Use transactions, outbox, retry, or compensation.
* **Service locator:** dependencies are hidden and failures appear at runtime. Construct them explicitly.
* **Over-abstraction:** every class has an interface with no variation. Remove accidental indirection.
* **Event drift:** consumers receive incompatible messages. Version schemas and test contracts.

## Testing Coupling

Unit-test policy against a port or fake, integration-test each adapter against the real boundary, and feature-test the wiring. A test that needs a full framework to check a value calculation signals unnecessary coupling. A unit test that mocks every SQL call gives weak evidence; test the repository with the real database.

Review dependency graphs, constructor size, shared state, and change history. If unrelated changes repeatedly touch one module, the coupling is evidence for a boundary or a simpler design.

## Exercises

1. Map dependencies for a checkout flow and classify each as data, temporal, deployment, or failure coupling.
2. Extract a narrow PaymentGateway port from a concrete provider and define its timeout and duplicate-charge behavior.
3. Find a service-locator call and replace it with explicit construction. Add a unit test with a fake dependency.
4. Decide whether an event or direct call is appropriate for sending a receipt after payment. Include recovery and observability.

## Review Questions

1. Why is zero coupling not a realistic goal?
2. Which dependencies deserve a port?
3. How do shared tables create deployment coupling?
4. Why are timeout and retry rules part of a dependency contract?
5. What does an event remove, and what new coupling does it create?
6. How can tests reveal accidental coupling?

## Summary

Coupling is necessary knowledge between components; accidental coupling makes change and failure expensive. Keep policy behind stable contracts, construct dependencies explicitly, translate data and errors at boundaries, design temporal and failure behavior, and use events only when independent timing or deployment justifies their cost.

## References

- [PHP Manual: Interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)
- [PHP-FIG PSR-11: Container Interface](https://www.php-fig.org/psr/psr-11/)
- [Martin Fowler: Inversion of Control](https://martinfowler.com/bliki/InversionOfControl.html)
- [Martin Fowler: Event-Driven](https://martinfowler.com/articles/201701-event-driven.html)

