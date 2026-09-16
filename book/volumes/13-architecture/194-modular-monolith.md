---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 194
title: Modular Monolith
slug: modular-monolith
status: complete
summary: ../../_ai/chapter-summaries/194-modular-monolith-summary.md
---

# Chapter 194 — Modular Monolith

## Why This Matters

A modular monolith is one deployable application divided into modules with explicit responsibilities and interfaces. It keeps in-process calls, one operational environment, and often one database while making ownership and dependencies visible. It is a useful middle ground when a codebase needs boundaries but does not yet need the cost of multiple services.

A monolith is modular only when modules cannot casually reach into one another's internals. Namespaces and folders help navigation; public APIs, dependency checks, and ownership rules create the boundary.

## Module Ownership

Define a module around a business capability such as Orders, Billing, or Inventory. Record:

* the state and invariants it owns;
* its public commands, queries, and events;
* its persistence tables and migration ownership;
* the actors and authorization rules;
* synchronous and asynchronous dependencies;
* operational metrics and data-retention policy.

A module may expose a small application API:

~~~php
<?php

declare(strict_types=1);

final readonly class PlaceOrder
{
    public function __construct(
        public int $customerId,
        public array $items,
        public string $requestKey,
    ) {
    }
}

interface Orders
{
    public function place(PlaceOrder $command): int;
}
~~~

Billing can call Orders::place without importing an order's ORM entity or writing its tables. The API should use stable commands, value objects, result DTOs, and versioned events rather than internal records.

## Enforcing Boundaries

Use separate namespaces and directory ownership, then enforce forbidden dependencies with static analysis, architecture tests, code review, and CI. A module's internal class should not be referenced from another module. If a query needs data from another module, call its public query or consume a projection.

Shared utility code can become a back door. Keep cross-cutting libraries small and stable; do not put business rules into a Common namespace simply to bypass ownership. A shared database table is also an API: define who owns columns and migrations, and prefer module-owned tables with explicit read models.

## Synchronous and Asynchronous Calls

An in-process call is simple and can participate in one transaction, but it couples latency, failure, and release timing. A domain event or outbox message decouples timing and allows a consumer to retry, but adds schema, delivery, duplication, and observability concerns.

Choose synchronously when the caller needs an immediate answer or consistency in one transaction. Choose asynchronously when work can be delayed, has independent scaling, or should not block the request. Use idempotency for commands and durable event IDs for consumers.

Do not use an event to hide a required invariant. If placing an order must reserve inventory before confirmation, model that consistency requirement explicitly. An eventual event after confirmation is appropriate only when the product accepts temporary divergence and recovery behavior.

## Database and Deployment

A single database does not require every module to query every table. Give each module schema ownership and expose read models or APIs for cross-module data. A transaction spanning many modules is a coupling signal; it may be correct, but its invariants and failure recovery must be explicit.

Deploy modules together in a monolith, but preserve compatibility while code and migrations roll forward or back. Use expand-and-contract changes, version event payloads, and keep old readers during migration. A module boundary should make a future extraction possible without making the current deployment unnecessarily distributed.

## Security and Operations

Tenant and authorization context must cross module boundaries. A privileged Orders module must not accept a caller's customer ID as proof of ownership. Carry a trusted actor or service identity, perform object authorization, and audit exceptional support access.

Measure per-module request volume, latency, errors, queue age, database time, and external dependency cost. Correlation IDs should follow synchronous calls and events. A module failure should produce a defined degraded result; avoid letting a convenience call silently grant access or discard a durable command.

## Failure and Threat Analysis

* **Internal reach-through:** another module edits owned tables. Enforce APIs and migration ownership.
* **Shared mutable model:** ORM entities leak business state across modules. Map DTOs and commands.
* **Event loss:** state commits but publication fails. Use an outbox and reconciliation.
* **Duplicate delivery:** consumers repeat effects. Use event IDs and idempotent handlers.
* **Distributed transaction illusion:** many modules appear atomic without a recovery plan. Define one transaction owner or a workflow.
* **Boundary bypass:** a CLI or job skips normal authorization. Re-check policy at the module operation.
* **Utility back door:** business logic enters shared helpers. Keep ownership explicit.

## Testing a Modular Monolith

Test each module through its public API and keep internal unit tests close to its implementation. Add architecture tests that fail when a module imports another module's internals. Integration-test module persistence and event serialization. Feature-test workflows that cross modules, including authorization, duplicate commands, and failure recovery.

Use contract tests for public module APIs so a refactor cannot silently change callers. Test event consumers with duplicate, old-version, malformed, and out-of-order messages where those conditions are possible.

## Exercises

1. Define module ownership for Orders, Billing, and Inventory, including tables, commands, events, and invariants.
2. Add a CI architecture check preventing Billing from importing Orders' internal persistence classes.
3. Design an order-placed outbox event and idempotent Billing consumer.
4. Identify a transaction crossing three modules and decide whether it should remain atomic or become a workflow.

## Review Questions

1. What makes a monolith modular?
2. Why is a shared database table an API?
3. When should a module call another synchronously?
4. What does an outbox solve, and what does it not solve?
5. Why should a future service extraction not force current network calls?
6. Which identity and authorization data must cross a module boundary?

## Summary

A modular monolith is one deployment with explicit business-module boundaries, owned state, narrow APIs, and controlled synchronous or asynchronous collaboration. Enforce ownership in code and CI, use outbox and idempotency for durable events, preserve migration compatibility, and test module contracts and cross-module failure behavior.

## References

- [Martin Fowler: MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- [Martin Fowler: Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- [Martin Fowler: Event-Driven](https://martinfowler.com/articles/201701-event-driven.html)
- [PHP Manual: Namespaces](https://www.php.net/manual/en/language.namespaces.php)

