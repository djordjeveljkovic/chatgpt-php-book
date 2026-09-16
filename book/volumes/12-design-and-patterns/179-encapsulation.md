---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 179
title: Encapsulation
slug: encapsulation
status: complete
summary: ../../_ai/chapter-summaries/179-encapsulation-summary.md
---

# Chapter 179 — Encapsulation

## Why This Matters

Encapsulation controls which code can observe or change a concept and forces changes through operations that preserve its invariants. Public fields and arrays make state easy to reach but leave every caller responsible for keeping it valid. One forgotten check can corrupt data or create a security boundary failure.

Encapsulation is not secrecy for its own sake. It is a way to make invalid states harder to construct, reduce the number of places that know representation details, and give a module a stable public contract.

## Invariants Behind a Boundary

Write the rule before choosing visibility. A bank account may not have a negative balance, a reservation's end must be after its start, and a document can be published only after approval. The object or aggregate that owns the rule should expose operations that enforce it.

~~~php
<?php

declare(strict_types=1);

final class Reservation
{
    private bool $cancelled = false;

    public function __construct(
        private readonly int $id,
        private readonly DateTimeImmutable $startsAt,
        private readonly DateTimeImmutable $endsAt,
    ) {
        if ($endsAt <= $startsAt) {
            throw new InvalidArgumentException('End must be after start');
        }
    }

    public function cancel(): void
    {
        if ($this->cancelled) {
            return;
        }

        $this->cancelled = true;
    }

    public function isCancelled(): bool
    {
        return $this->cancelled;
    }
}
~~~

Callers cannot set cancelled to an arbitrary value or replace an endpoint without going through the policy. Idempotent cancel behavior is deliberate; another domain may need an exception for a repeated cancellation. The public methods define the contract.

## Commands, Queries, and Collections

A query can expose information without exposing mutable storage. Return a scalar, value object, immutable snapshot, or copy rather than an internal mutable collection. A command should name the business operation, not merely expose a setter such as setStatus.

PHP's private visibility is a language boundary within a class, not a complete security boundary. Reflection, debugging tools, serialization, and database access still require operational controls. Encapsulation in application code must be reinforced with database constraints, authorization, and input validation.

## Aggregates and Persistence

An aggregate groups state whose invariants must be changed together. Its repository can persist a snapshot, but persistence code should not bypass domain operations for ordinary changes. Hydration of legacy or corrupted data needs an explicit policy: validate it, quarantine it, or provide a carefully controlled reconstitution path.

Do not return an ORM entity with every property publicly writable just because a mapper is convenient. Use commands for allowed changes and map persistence records into domain objects. A database constraint is a second line of defense for uniqueness, foreign keys, and non-null rules.

## Encapsulation at Module Boundaries

A module should expose a small interface and keep its algorithms, storage schema, and provider details private. Namespaces and interfaces help communicate the boundary; they do not stop callers from depending on public arrays or vendor types. Return stable DTOs and value objects at the boundary.

Avoid “getter and setter” classes that expose every field without a policy. A getter is reasonable when the value is part of the contract; a setter is a design question. Replace setState('approved') with approve(actor, reason) when the transition has authorization or audit rules.

## Security and Failure

Encapsulation protects security-relevant state only when all entry points use it. A queue consumer, import script, or admin command that writes directly to a table can bypass the same invariant enforced by an HTTP service. Put domain rules below transport and reuse the operation at every entry point.

When an operation fails halfway, the boundary must define transaction and recovery behavior. An in-memory object may be valid while persistence fails. Use an explicit unit of work, transaction, outbox, or retry policy; do not pretend private fields provide atomicity.

## Failure and Threat Analysis

* **Public mutable state:** callers create invalid combinations. Use commands and invariant checks.
* **Setter sprawl:** every state transition becomes legal. Name operations and enforce preconditions.
* **Bypassed aggregate:** a job or migration changes rows directly. Protect with constraints and shared domain operations.
* **Leaky DTO:** internal fields become a public contract. Map an allow-list.
* **Invalid hydration:** corrupted data enters a valid object. Validate and quarantine or reconstitute explicitly.
* **False security:** private visibility is treated as authorization. Enforce subject and tenant policy separately.
* **Partial update:** memory and database diverge. Define transaction and recovery behavior.

## Testing Encapsulation

Test public operations and invariants, not private fields. Test illegal construction, valid transitions, repeated operations, authorization context, and failure recovery. Integration-test persistence mapping and constraints. A test that uses reflection to set private state usually signals a missing public scenario or a deliberate reconstitution API.

Use mutation testing to check that important guards are detected. Contract tests at module boundaries protect callers while internal representation changes.

## Exercises

1. Refactor a public array of order items into an encapsulated collection with add, remove, and total operations.
2. Model a document with draft, approved, and published states. Define public transitions and illegal transitions.
3. Design a reconstitution path for legacy data that violates a new invariant.
4. Find a queue or import path that bypasses a domain operation and add a regression test.

## Review Questions

1. What problem does encapsulation solve beyond private visibility?
2. Why are setters often weaker than named domain operations?
3. What belongs in an aggregate?
4. Why must database constraints reinforce object invariants?
5. How can hydration bypass encapsulation?
6. Why is encapsulation not authorization?

## Summary

Encapsulation protects invariants by restricting state changes to meaningful operations and exposing a stable boundary. Keep mutable collections and representations private, separate commands from queries, map persistence explicitly, enforce rules across every entry point, and pair object boundaries with authorization, constraints, and recovery design.

## References

- [PHP Manual: Visibility](https://www.php.net/manual/en/language.oop5.visibility.php)
- [PHP Manual: Readonly Properties](https://www.php.net/manual/en/language.oop5.properties.php#language.oop5.properties.readonly)
- [Martin Fowler: Aggregate](https://martinfowler.com/bliki/DDD_Aggregate.html)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

