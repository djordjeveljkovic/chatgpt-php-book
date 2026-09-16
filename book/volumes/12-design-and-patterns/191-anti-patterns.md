---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 191
title: Anti-Patterns
slug: anti-patterns
status: complete
summary: ../../_ai/chapter-summaries/191-anti-patterns-summary.md
---

# Chapter 191 — Anti-Patterns

An anti-pattern is a repeated solution that appears reasonable but creates predictable harm in a particular context. It is not a list of forbidden syntax. A singleton, repository, inheritance hierarchy, or microservice can be appropriate under constraints and harmful when copied without those constraints.

The engineering response is to identify the cost, make it measurable, and change the smallest boundary that reduces it. Naming an anti-pattern should improve a design conversation; it should not replace one.

## Why this matters

A team can add abstractions, tests, queues, and services while making a system harder to change and less reliable. The symptoms are often familiar: every feature touches the same class, tests assert implementation details, production behavior depends on global state, or a simple request crosses six network boundaries.

Use evidence before refactoring. Look at change frequency, defect patterns, latency, memory, deployment failures, coupling, and the cost of explaining a component to a new engineer. A smell suggests a question; it does not prove a diagnosis.

## The god class and the smart controller

A god class owns unrelated policies, data access, formatting, transport, and side effects. A smart controller is the web-shaped version:

```php
final class CheckoutController
{
    public function post(): Response
    {
        $cart = $this->db->query('SELECT ...');
        // Validate fields, calculate tax, call payment provider,
        // send email, write audit rows, and render HTML here.
    }
}
```

The problem is not the number of lines by itself. The class has multiple reasons to change and no clear boundary for testing or authorization. Extract cohesive policies and adapters while preserving a thin application flow:

```php
final class CheckoutHandler
{
    public function __construct(
        private CartService $carts,
        private PaymentAuthorizer $payments,
        private OrderWriter $orders,
    ) {
    }

    public function handle(CheckoutCommand $command): OrderView
    {
        $cart = $this->carts->ownedBy($command->cartId, $command->customerId);
        $authorization = $this->payments->authorize($cart->total(), $command->requestKey);

        return $this->orders->create($cart, $authorization);
    }
}
```

The handler still coordinates a use case. It should not become a new god class with every business rule pushed into it. Extract only when a responsibility has a name, invariant, or independent test boundary.

## Abstraction soup and ceremony

An interface, repository, service, manager, factory, DTO, mapper, and wrapper for every table can create indirection without protecting a real boundary. “Service” often becomes a name for any code that did not fit elsewhere. The reader must follow many files to understand one operation.

Prefer a concrete class or function when there is one stable implementation and no meaningful boundary. Add an interface where a capability has alternate implementations, an external protocol needs isolation, or a test needs a controlled boundary. Keep the contract small and use the domain vocabulary. [Chapter 185 — Dependency Injection](./185-dependency-injection.md) and [Chapter 186 — Abstraction](./186-abstraction.md) describe the tradeoffs.

A repository that merely forwards every ORM method can be needless ceremony. A repository that protects tenant scope, query shape, transaction rules, or a domain invariant may be valuable. Judge it by the decision it centralizes.

## Global state and the service locator

Static registries, mutable singletons, and container lookups inside business methods hide inputs and lifetimes:

```php
final class ReportService
{
    public function run(): Report
    {
        return App::container()->get(ReportRepository::class)->build();
    }
}
```

Tests need hidden setup, workers can leak state between jobs, and a caller cannot see what the service requires. Resolve the object at the composition root and inject the narrow dependency. If a resource must be shared, inject one deliberately shared instance and document its lifetime. [Chapter 185](./185-dependency-injection.md) covers composition roots and container boundaries.

## Inheritance for reuse

Inheritance is often chosen to reuse a method, then grows into a fragile hierarchy. A subclass inherits lifecycle assumptions, protected state, and methods it should not expose. A base class change can alter every subclass.

Use composition when the relationship is “has a capability” rather than “is substitutable for.” Use an interface for a capability and a decorator or collaborator for optional behavior. Inheritance is appropriate when the subtype genuinely preserves the base contract and the base class owns meaningful invariant-preserving behavior.

A test that must call protected methods or disable a parent constructor is evidence to inspect the hierarchy. Refactor at a stable seam instead of adding more override hooks.

## Primitive obsession and boolean blindness

Strings and integers can represent incompatible concepts. A method with several booleans is particularly difficult to read:

```php
$report->deliver($userId, true, false, true);
```

The call site does not explain the policy, and adding another mode creates more combinations. Use value objects, enums, or an options object with named fields:

```php
enum DeliveryChannel: string
{
    case Email = 'email';
    case Download = 'download';
}

final readonly class DeliveryOptions
{
    public function __construct(
        public DeliveryChannel $channel,
        public bool $includeAttachments,
        public bool $notifyOwner,
    ) {
    }
}
```

Keep validation at construction and make invalid combinations explicit. Do not turn every scalar into a class; a type is worthwhile when it carries a rule, unit, identity, or meaningful operation.

## Copy and paste and shotgun surgery

Duplicated validation, authorization, SQL, and retry logic drift. A change then requires edits in many files, and one missed edit creates inconsistent behavior. Search by behavior, not only by symbol, and identify the invariant being repeated.

Extract a cohesive function, policy, or adapter when the duplicated rule has one owner. Avoid a giant utility class that becomes a new global dumping ground. Tests should assert the invariant through each important entry point so a shared extraction does not accidentally bypass a boundary.

## Premature distribution and speculative flexibility

Splitting a small application into services adds network failure, deployment coordination, observability, serialization, and data consistency costs. Adding plugin systems, generic workflows, or five interchangeable implementations before a requirement exists creates maintenance work and delays feedback.

Start with a modular monolith when one process and one deployment meet the constraints. Create a network boundary when independent scaling, ownership, isolation, or failure behavior justifies it. Design the contract, idempotency, timeout, authentication, and data ownership before moving code across a network.

Likewise, optimize a measured bottleneck. A cache can create invalidation and consistency problems; a clever data structure can add complexity without improving the actual workload. Profile and record the constraint before introducing a performance mechanism.

## Test smells as design signals

A test suite can expose anti-patterns:

- every test mocks half the object graph;
- tests require global reset hooks and order-dependent fixtures;
- a private method is tested instead of observable behavior;
- assertions duplicate the implementation's algorithm;
- integration tests use a different database or transport than production;
- tests are slow because unrelated infrastructure is constructed for every unit.

Do not fix a test smell by weakening assertions blindly. Decide whether the production boundary is wrong, the test level is wrong, or the behavior is genuinely an interaction contract. Mocks, stubs, fakes, and spies have different purposes; see [Chapter 166 — Mocks](../11-testing/166-mocks.md) through [Chapter 169 — Spies](../11-testing/169-spies.md).

## Refactoring with safety

Refactor incrementally:

1. Capture the current behavior with characterization or integration tests.
2. Name the cost and choose a measurable goal.
3. Introduce one seam at a time.
4. Move behavior without changing the contract.
5. Run focused and broad tests, static analysis, and performance checks.
6. Remove the old path and record the new ownership.

Keep deployment and rollback in the plan for changes to transactions, queues, schemas, authorization, and external contracts. A refactor that passes unit tests but changes retry or consistency behavior is a production behavior change.

## Exercises

1. Choose a god controller and list its reasons to change. Extract one adapter or policy while preserving its endpoint behavior.
2. Find a method with three booleans. Replace them with an enum and an options value object, then test invalid combinations.
3. Review a proposed microservice split. List the network, data, deployment, and observability costs and identify the constraint that justifies each.
4. Select a mock-heavy test and decide whether it needs a fake, an integration test, or a smaller production boundary.

## Review questions

- Why is an anti-pattern contextual rather than an absolute rule?
- What evidence distinguishes a god class from a cohesive coordinator?
- When is a repository useful and when is it ceremony?
- Which costs does a network boundary introduce?
- How can test smells reveal hidden production coupling?
- What should be measured before a refactor or optimization?

## Summary

Anti-patterns are recurring responses whose costs exceed their value in a given context. Diagnose with evidence, reduce hidden state and accidental coupling, keep abstractions and boundaries purposeful, and refactor in small tested steps. Prefer clarity and explicit tradeoffs over pattern names or architectural fashion.

## References

- [Martin Fowler: Code Smell](https://martinfowler.com/bliki/CodeSmell.html)
- [Martin Fowler: Design Stamina Hypothesis](https://martinfowler.com/bliki/DesignStaminaHypothesis.html)
- [Martin Fowler: Refactoring](https://martinfowler.com/books/refactoring.html)
- [PHP manual: Classes and Objects](https://www.php.net/manual/en/language.oop5.php)
