---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 178
title: Cohesion
slug: cohesion
status: complete
summary: ../../_ai/chapter-summaries/178-cohesion-summary.md
---

# Chapter 178 — Cohesion

## Why This Matters

Cohesion describes how strongly the responsibilities inside a module belong together. A cohesive component has one reason for its rules, data, and changes to be near each other. Low cohesion produces “utility” classes and services that know unrelated policies, making names vague, dependencies broad, and tests noisy.

High cohesion is about a meaningful boundary, not a small line count. A module can contain several operations when they share a domain concept and change together.

## Responsibility and Change

Group code by the decisions it owns. An invoice component may calculate totals, apply tax rules, and expose a total because those rules share the invoice concept. Email formatting, payment capture, and PDF storage change for different reasons and belong at separate boundaries.

The Single Responsibility Principle is a change heuristic: a class should have one reason to change. It does not mean every method gets a class. Ask which stakeholders, policies, data, and failure modes would require a modification. If unrelated answers appear, cohesion is weak.

## A Cohesive Domain Object

Put an invariant near the data it protects. A money value object can own currency compatibility and arithmetic rather than letting every caller repeat slightly different checks:

~~~php
<?php

declare(strict_types=1);

final readonly class Money
{
    public function __construct(
        public int $cents,
        public string $currency,
    ) {
        if ($cents < 0 || !preg_match('/^[A-Z]{3}$/', $currency)) {
            throw new InvalidArgumentException('Invalid money value');
        }
    }

    public function add(self $other): self
    {
        if ($other->currency !== $this->currency) {
            throw new InvalidArgumentException('Currencies differ');
        }

        return new self($this->cents + $other->cents, $this->currency);
    }
}
~~~

The object has one cohesive responsibility: a valid monetary value and its arithmetic. It does not send invoices, call a currency API, or persist itself. Those concerns can use Money without knowing its representation rules.

## Cohesion and Module Boundaries

A module boundary should make common changes local. Organize around a domain capability when teams and requirements change by capability; organize around technical layers when the system is small or the layer genuinely owns a stable concern. Neither folders nor namespaces create cohesion by themselves.

Look at dependency direction. A cohesive module exposes a small public surface and keeps internal rules private. If every caller reaches into five internal classes, the module is a collection of files rather than a boundary. If one “UserService” handles registration, billing, exports, password reset, and support impersonation, split by decision and lifecycle rather than by arbitrary method count.

## Cohesion, State, and Transactions

Data that must change atomically should usually be modeled near the operation that enforces its invariant. A reservation's interval and status may belong in one transaction; a reporting projection can be separate and eventually consistent. Splitting data only because tables or classes look large can move a consistency rule into fragile coordination code.

A cohesive command handler can validate a command, call a domain operation, and persist the result within an explicit transaction boundary. It should not also own HTTP serialization, SQL dialect details, and email templates. Keep each layer's responsibility clear while preserving the domain operation's cohesion.

## Finding Low Cohesion

Warning signs include:

* a class name ending in Manager, Helper, or Util with unrelated methods;
* many dependencies used by only one method;
* frequent changes from unrelated teams;
* tests with large fixtures and many mocks;
* methods that share no data or policy;
* conditional branches selecting unrelated modes;
* a module whose public API exposes internal data structures.

Refactor by first naming the responsibilities and their shared invariants. Move a coherent group behind a clear interface, leave a delegating seam, and let tests protect behavior. Avoid splitting a cohesive concept merely to satisfy a numeric class-size rule.

## Failure and Threat Analysis

* **Anemic domain:** rules live in many callers and diverge. Keep invariants with the concept that owns them.
* **God service:** one class coordinates unrelated workflows. Split by policy and lifecycle.
* **False cohesion:** methods share a noun but not a reason to change. Examine stakeholders and failure modes.
* **Transaction split:** related state changes commit separately. Keep the invariant in one transaction or use an explicit workflow.
* **Leaky module:** callers mutate internal collections. Expose commands or immutable views.
* **Premature extraction:** tiny changes become many abstractions. Wait for a real boundary and evidence.

## Testing Cohesion

Test value objects and domain policies as units. Test module boundaries through public commands and queries rather than internal classes. A cohesive module should allow fixtures to focus on one concept and should not require unrelated providers.

Use change history and test setup as evidence. If one class's tests need authentication, PDF rendering, a payment sandbox, and a queue, its responsibilities likely span several cohesive boundaries. Refactoring should preserve contract tests while moving internal implementation.

## Exercises

1. Take a service with at least five methods and group them by shared policy, data, and change reason.
2. Design a cohesive Money or DateRange value object with invariants and tests.
3. Identify a transaction rule split across two modules. Decide whether to move it or define an explicit workflow.
4. Review a module's public API and remove one leaked internal data structure.

## Review Questions

1. How does cohesion differ from class size?
2. What is a “reason to change” in the Single Responsibility Principle?
3. Why should an invariant live near the data it protects?
4. Which signs indicate a low-cohesion service?
5. How do module tests reveal cohesion problems?
6. When can technical-layer organization be appropriate?

## Summary

Cohesion keeps related data, rules, and changes together around a meaningful domain responsibility. Group by shared invariants and change reasons, keep transactions with the state they protect, expose narrow module APIs, and refactor low-cohesion services by policy and lifecycle rather than by arbitrary size.

## References

- [PHP Manual: Classes and Objects](https://www.php.net/manual/en/language.oop5.php)
- [Robert C. Martin: The Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html)
- [Martin Fowler: Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)

