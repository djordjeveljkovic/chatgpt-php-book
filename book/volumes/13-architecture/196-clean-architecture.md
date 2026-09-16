---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 196
title: Clean Architecture
slug: clean-architecture
status: complete
summary: ../../_ai/chapter-summaries/196-clean-architecture-summary.md
---

# Chapter 196 — Clean Architecture

## Why This Matters

Clean Architecture is a way to keep business rules independent from delivery mechanisms and infrastructure. It arranges policy toward the center and details toward the edge, with dependencies pointing inward. The benefit is not a fixed number of rings; it is the ability to change a framework, database, or transport without rewriting the rules that matter.

Clean Architecture overlaps with layered and hexagonal architecture. Use the name only when the dependency rule and use-case boundaries are actually enforced.

## The Dependency Rule

A practical PHP system can have:

* **Entities:** domain concepts and invariants.
* **Use cases:** application-specific operations and policies.
* **Interface adapters:** controllers, presenters, repositories, and gateways translating between models.
* **Frameworks and drivers:** HTTP framework, PDO/ORM, queue, filesystem, and provider SDK.

Dependencies point toward entities and use cases. Inner code does not import outer code. A use case can define a port for persistence; an outer adapter implements it.

~~~php
<?php

declare(strict_types=1);

final readonly class CreateInvoice
{
    public function __construct(
        public int $accountId,
        public int $amountCents,
        public string $currency,
    ) {
    }
}

interface InvoiceStore
{
    public function save(CreateInvoice $command): int;
}

final class CreateInvoiceUseCase
{
    public function __construct(private InvoiceStore $invoices)
    {
    }

    public function execute(CreateInvoice $command): int
    {
        if ($command->amountCents <= 0) {
            throw new InvalidArgumentException('Amount must be positive');
        }

        return $this->invoices->save($command);
    }
}
~~~

The use case knows the capability it needs, not PDO or an ORM. An HTTP controller can turn JSON into CreateInvoice; a CLI adapter can construct the same command; a test can provide an in-memory InvoiceStore.

## Models at Boundaries

Avoid one model for every boundary. A request DTO represents client input, a use-case command represents an authorized operation, an entity protects domain state, and a response DTO represents a public contract. Reusing a database row as all four creates accidental coupling and mass-assignment risk.

Mapping is deliberate work. Validate and authorize before executing a command, map domain results to safe output, and translate infrastructure failures to stable application outcomes. The inner model should not contain framework annotations or serialize itself into an HTTP response.

## Use Cases and Transaction Policy

A use case coordinates a business operation. It may validate cross-entity rules, call a repository port, record an event, and define the transaction boundary. Keep transaction mechanics in an outer implementation, but make atomicity visible in the port or unit-of-work contract.

Do not call a remote provider inside a database transaction without a recovery plan. Use a state transition, outbox, provider idempotency key, and reconciliation workflow. Clean boundaries expose distributed failure; they do not make it disappear.

## Practical Package Structure

A project might use:

~~~text
src/
  Domain/
  Application/
    Invoice/
  Adapters/
    Http/
    Persistence/
    Providers/
  Infrastructure/
  Bootstrap/
~~~

A vertical feature structure can work equally well if it preserves the dependency rule. Use namespaces and static analysis to prevent Application importing Infrastructure or Domain importing a framework. Architecture tests should fail on forbidden dependencies before a reviewer has to notice them.

## When Clean Architecture Helps

It is valuable when domain rules are important, infrastructure changes, multiple entry points share use cases, teams need clear ownership, or testing an adapter separately reduces risk. It may be excessive for a small CRUD screen with no meaningful domain behavior. Rings and abstractions add mapping, construction, and cognitive costs.

Start with a clear use case and a domain invariant. Extract ports when a boundary needs substitution or independent failure policy. Keep the outer framework thin. The architecture should make a changed database or HTTP framework a localized change, not force every feature through ceremonial interfaces.

## Failure and Security

Authentication can happen in an outer adapter, but authorization of the actual resource and operation belongs in the use-case policy. A command from a queue or CLI must carry a trusted actor context or use a service identity; do not trust account IDs in a payload.

Keep secrets, tokens, SQL, stack traces, and provider response bodies out of inner models and public errors. Apply input size limits at the outer boundary, domain invariants inside, and database constraints at the persistence boundary. Use correlation IDs and safe audit events without leaking personal data.

## Testing Clean Architecture

Unit-test entities and use cases through their public APIs with controlled ports. Integration-test adapters against PDO, queues, filesystems, and providers. Feature-test controllers, middleware, authentication, serialization, and error mapping. Add architecture checks for dependency direction.

Tests should not assert that a framework was called when the use-case contract is the behavior. Conversely, a passing use-case unit test cannot prove an adapter serializes a date or binds SQL correctly.

## Failure and Threat Analysis

* **Ring theater:** folders imply layers while imports violate the rule. Enforce dependencies in CI.
* **Over-mapping:** every trivial field gets a ceremonial DTO. Keep boundaries proportional to risk.
* **Framework leakage:** inner code depends on request, ORM, or annotations. Map at adapters.
* **Transaction ambiguity:** use case changes multiple stores without atomicity policy. Define unit of work or workflow.
* **Authorization at edge only:** a worker bypasses HTTP middleware. Authorize at use-case entry.
* **Port bloat:** an interface mirrors infrastructure. Define the use case's required capability.
* **Stale command:** queued data no longer matches current policy. Include version and revalidate on handling.

## Exercises

1. Draw Clean Architecture rings for an invoice feature and list which dependencies point inward.
2. Extract a use-case port from an ORM repository and write an adapter integration test.
3. Define a command, entity, and response DTO for a multi-tenant update. Mark where validation and authorization occur.
4. Add an architecture test or static-analysis rule preventing Domain from importing framework code.

## Review Questions

1. What is the dependency rule?
2. How do entities, use cases, adapters, and drivers differ?
3. Why should a database row not be every boundary's model?
4. Where should authorization occur for a queued command?
5. What costs does Clean Architecture introduce?
6. Which integration tests remain necessary after use-case unit tests pass?

## Summary

Clean Architecture points dependencies inward toward entities and use cases while keeping frameworks, databases, and providers at the edge. Use separate models where contracts differ, define narrow ports, make transaction and authorization policies explicit, enforce dependency rules, and test inner behavior and outer adapters at their true boundaries.

## References

- [Robert C. Martin: The Clean Code Blog](https://blog.cleancoder.com/)
- [Robert C. Martin: Clean Architecture](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/)
- [Alistair Cockburn: Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [PHP Manual: Namespaces](https://www.php.net/manual/en/language.namespaces.php)

