---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 193
title: Layered Architecture
slug: layered-architecture
status: complete
summary: ../../_ai/chapter-summaries/193-layered-architecture-summary.md
---

# Chapter 193 — Layered Architecture

## Why This Matters

Layered architecture separates responsibilities so a change in presentation, domain policy, or infrastructure does not spread through every file. A common PHP arrangement has presentation, application, domain, and infrastructure layers. The value comes from dependency direction and contracts, not from directory names.

Layers are boundaries with rules. If every layer can call every other layer, the project has folders but no architecture.

## The Layers

A practical arrangement is:

* **Presentation:** HTTP, CLI, serialization, authentication middleware, and transport errors.
* **Application:** use cases, transaction coordination, authorization context, and ports needed by the use case.
* **Domain:** entities, value objects, policies, and invariants independent of transport and infrastructure.
* **Infrastructure:** PDO repositories, framework adapters, filesystems, queues, and external providers.

A request flows inward to a use case and outward through a presenter or adapter:

~~~text
controller -> application command -> domain operation
                                     |
                         repository port / transaction
                                     v
                              infrastructure
~~~

The application layer can depend on domain types and ports. Infrastructure implements ports. Presentation should not query tables or construct vendor SDK requests directly.

## A Use Case Boundary

~~~php
<?php

declare(strict_types=1);

final readonly class RegisterAccount
{
    public function __construct(
        public string $email,
        public string $passwordHash,
    ) {
    }
}

interface AccountWriter
{
    public function insert(RegisterAccount $command): int;
}

final class RegisterAccountHandler
{
    public function __construct(private AccountWriter $accounts)
    {
    }

    public function handle(RegisterAccount $command): int
    {
        return $this->accounts->insert($command);
    }
}
~~~

The controller can parse JSON, validate content type, authenticate the request context, construct a command, and map the returned ID to a response. It does not need to know whether AccountWriter uses PDO, an ORM, or a remote service.

## Dependency Direction and Leakage

A strict dependency rule keeps lower-level policy independent of higher-level details. In PHP, namespaces and interfaces communicate the rule, while static analysis and code review can enforce forbidden imports. Do not import a framework Request into a domain entity or pass a PDO statement into a business policy.

Leakage often begins as convenience:

* a repository returns an ORM model that controllers serialize directly;
* a domain service accepts an associative HTTP payload;
* an infrastructure exception becomes a public response;
* a template reads a database row and invents business rules.

Translate at boundaries. The translation is intentional cost that prevents internal representation from becoming a public contract.

## Transactions and Cross-Layer Work

The application layer usually owns the use-case transaction because it knows which domain changes and outbox records must commit together. The infrastructure transaction adapter owns PDO calls and isolation details; the domain should not call beginTransaction itself.

If a use case invokes an external provider, do not hold a database transaction open across an unbounded network call. Use an explicit workflow, reservation state, outbox, provider idempotency key, and reconciliation. The layer boundary must not hide a distributed transaction that the system cannot guarantee.

## Variations and Trade-offs

Some systems use presentation, application, domain, and infrastructure as strict layers. Others use a simpler three-layer design or vertical slices that contain all layers per feature. Strict layering can reduce accidental dependencies but create pass-through classes and mapping overhead. A vertical slice can improve feature locality but requires discipline to preserve shared policy and module boundaries.

Choose the direction that matches change and ownership. Enforce one-way dependencies where they protect domain stability, but do not force every trivial query through five classes.

## Failure and Security

Authorization belongs at the use-case boundary with the actual actor and resource. Presentation middleware can authenticate, but a controller-only check can be bypassed by a queue or CLI path. Input validation begins at presentation and domain invariants remain below it.

Define error mapping: validation problems become safe client responses; unique constraint violations become a documented conflict; timeouts become retryable or degraded outcomes; unexpected exceptions become generic failures with correlation IDs. Do not leak SQL, stack traces, credentials, or provider details.

## Testing Layered Systems

Unit-test domain policies and application handlers with ports or fakes. Integration-test infrastructure adapters against the real database and provider protocol. Feature-test presentation wiring, middleware, serialization, and authorization. Add static architecture checks for forbidden imports if the dependency rule is important.

A test that crosses layers is not automatically better. Test each boundary's contract and add a full-flow test for the risky user behavior.

## Exercises

1. Map an existing PHP application into presentation, application, domain, and infrastructure responsibilities.
2. Extract an application command from a controller and define its port for persistence.
3. Identify three framework or ORM types leaking into a domain class and replace them with domain types.
4. Design transaction and outbox behavior for a layered order-creation use case.

## Review Questions

1. What makes layers architectural rather than merely folders?
2. Which layer should coordinate a use-case transaction?
3. Why should a domain object not receive an HTTP request or PDO statement?
4. What are the costs of strict layering?
5. How should a timeout cross a layer boundary?
6. Which tests belong at each layer?

## Summary

Layered architecture separates presentation, application, domain, and infrastructure while enforcing dependency direction. Keep transport and vendor details at the edges, coordinate transactions in the application boundary, map errors safely, test each contract at its real boundary, and use strict layering only where its protection outweighs mapping cost.

## References

- [PHP Manual: Namespaces](https://www.php.net/manual/en/language.namespaces.php)
- [PHP-FIG PSR-11: Container Interface](https://www.php-fig.org/psr/psr-11/)
- [Martin Fowler: Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- [Martin Fowler: Layering](https://martinfowler.com/architecture/layering.html)

