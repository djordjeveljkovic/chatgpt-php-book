---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 192
title: Simple Architecture
slug: simple-architecture
status: complete
summary: ../../_ai/chapter-summaries/192-simple-architecture-summary.md
---

# Chapter 192 — Simple Architecture

## Why This Matters

Architecture is the set of boundaries and decisions that determine how a system changes, runs, fails, and is operated. A small PHP application does not need a ceremony-heavy framework of abstractions. It does need clear ownership of input parsing, domain rules, persistence, side effects, and error handling.

Start with the simplest structure that makes the current risks visible. Add a boundary when a requirement, failure mode, team boundary, or deployment constraint justifies it. Architecture should reduce change cost and make important behavior easier to verify.

## A Small Request Path

A simple application can have a front controller, application service, repository, and database adapter:

~~~text
HTTP request
  -> controller parses and authorizes
  -> application service coordinates a use case
  -> domain objects enforce invariants
  -> repository persists
  -> response maps the result
~~~

The layers are roles, not necessarily directories. Keep transport types out of domain rules and keep SQL details out of the controller. A single codebase can still have explicit dependencies.

~~~php
<?php

declare(strict_types=1);

final readonly class CreateNote
{
    public function __construct(
        public int $authorId,
        public string $body,
    ) {
        if (trim($body) === '') {
            throw new InvalidArgumentException('Note body is required');
        }
    }
}

interface NoteRepository
{
    public function add(CreateNote $command): int;
}

final class CreateNoteService
{
    public function __construct(private NoteRepository $notes)
    {
    }

    public function execute(CreateNote $command): int
    {
        return $this->notes->add($command);
    }
}
~~~

The example is intentionally small. An application service is useful when it coordinates a use case, transaction, authorization context, and side effects. A one-line wrapper around a repository may be unnecessary; keep the direct call until another responsibility earns the boundary.

## Composition and Explicitness

Construct the graph in one place. A composition root can create a PDO connection, repository, service, and controller, then pass them explicitly. This makes lifetimes and substitutes visible in tests. A service locator or global container hides dependencies and causes failures to appear at runtime.

Keep configuration separate from domain data. Parse environment values at startup, validate them, and inject typed settings. Do not let business code read environment variables in every method.

## Choosing Simplicity

A simple architecture is not an excuse for a giant controller or shared mutable globals. Keep each operation readable, name domain invariants, and isolate volatile boundaries. A small application may use a single database and synchronous email while still using an outbox for a critical event.

Use a direct function or class when it is clear. Introduce repositories, interfaces, and modules when they protect a real variation or boundary:

* a provider has a timeout and failure taxonomy;
* a database query needs integration tests and transaction policy;
* a command is reused by HTTP and a worker;
* two teams need a stable contract;
* a feature has independent state and authorization rules.

Avoid abstractions whose only purpose is to make a framework call look different. Every interface has a maintenance cost: implementations, wiring, tests, and versioning.

## Failure and Operations

Write the failure path beside the success path. Define behavior for invalid input, missing rows, duplicate commands, database unavailability, provider timeout, and a process crash after commit. Set network and database timeouts, log safe correlation data, and expose health signals.

A simple architecture can still be durable. Use database constraints for invariants, transactions for atomic changes, idempotency keys for harmful retries, and outbox records when publication must follow a commit. Simplicity means fewer moving parts with explicit behavior, not fewer safeguards.

## Growth Without Premature Complexity

When a feature grows, first add tests and characterize the existing contract. Extract a cohesive domain object or application service, then move infrastructure behind a port only if its behavior varies or needs isolation. Keep deployment and rollback safe while changing the structure.

Architecture evolves by evidence: change frequency, defect patterns, dependency latency, team ownership, and operational incidents. Do not adopt a distributed architecture because a diagram looks advanced. A modular monolith or a layered structure may provide the needed boundaries with less operational cost.

## Failure and Threat Analysis

* **Giant controller:** parsing, policy, SQL, and email become inseparable. Move one use case behind a service and test it.
* **Global state:** requests and tests influence one another. Inject configuration, clocks, and stores.
* **Leaky domain:** database rows and HTTP arrays become business APIs. Map at the boundary.
* **Missing transaction:** a partial failure leaves inconsistent state. Define atomicity and recovery.
* **Premature abstraction:** interfaces multiply without a real boundary. Remove or defer them.
* **Unbounded dependency:** a provider call consumes workers. Set deadlines, retries, and circuit policy.
* **Unscoped access:** a simple lookup crosses a tenant boundary. Authorize at the operation and query.

## Testing Simple Architecture

Use unit tests for domain rules, integration tests for repositories and database constraints, and feature tests for routing and authorization. The small structure should make each test level obvious. When one test needs every service and external provider to verify a pure calculation, the design is coupling unrelated concerns.

Review a dependency graph and a failure matrix before adding a new abstraction. The best simple architecture is the one the team can explain, test, deploy, and recover under pressure.

## Exercises

1. Draw the request path for a small PHP feature and mark transport, application, domain, persistence, and side-effect responsibilities.
2. Refactor a controller that parses input, writes SQL, and sends email into a simple explicit service.
3. List the failure points in a create-order request and choose transactions, idempotency, or an outbox where appropriate.
4. Identify one abstraction in an application that has no independent policy. Remove or justify it.

## Review Questions

1. What makes a simple architecture reliable rather than merely small?
2. When does an application service earn its boundary?
3. Why should configuration be parsed at startup?
4. Which safeguards still belong in a simple application?
5. What evidence justifies introducing a new architectural layer?
6. How can a controller reveal too many responsibilities?

## Summary

Start with a small, explicit request path and add boundaries when risk, change, ownership, or failure behavior requires them. Keep transport, domain, persistence, and side effects distinct enough to test, inject dependencies explicitly, define failure recovery, and prefer evidence over architectural fashion.

## References

- [PHP Manual: Dependency Injection by type declarations](https://www.php.net/manual/en/language.oop5.type-declarations.php)
- [Martin Fowler: Is Design Dead?](https://martinfowler.com/articles/designDead.html)
- [Martin Fowler: YAGNI](https://martinfowler.com/bliki/Yagni.html)

