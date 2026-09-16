---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 33
title: Final
slug: final
status: complete
summary: ../../_ai/chapter-summaries/033-final-summary.md
---

# Chapter 33 — Final

## Why This Matters

Inheritance is useful when a child really is a substitutable specialization of its parent. It is dangerous when it is only being used to reuse a few lines of code. Every non-final extension point becomes part of the class's contract: somebody can override it, call it from a subclass, depend on its protected state, or rely on an accidental ordering of hooks.

The `final` keyword lets an API make that contract explicit. It can close a class, a method, a property, or a class constant against a particular kind of change. This is not a style contest between “open” and “closed” code. It is a decision about which variation the design can safely support.

## Mental Model

Think of inheritance as a boundary with two directions:

```text
parent promises behavior ──▶ child may specialize what remains open
parent closes behavior   ──▶ child must use the invariant as supplied
```

`final` does not make an object immutable, and it does not prevent composition. It restricts inheritance-time replacement. A final method can still call collaborators, read mutable state, and produce different results based on those collaborators. A final class can still implement interfaces and receive dependencies.

## Core Concept

```php
<?php

declare(strict_types=1);

final class ReservationId
{
    public function __construct(public readonly string $value)
    {
        if ($value === '') {
            throw new InvalidArgumentException('An id cannot be empty.');
        }
    }
}

interface ReservationRepository
{
    public function save(Reservation $reservation): void;
}

final class ReservationService
{
    public function __construct(private ReservationRepository $repository)
    {
    }

    final public function reserve(Reservation $reservation): void
    {
        $this->validate($reservation);
        $this->repository->save($reservation);
    }

    private function validate(Reservation $reservation): void
    {
        // Domain invariant: validation precedes persistence.
    }
}
```

`final class ReservationService` prevents a subclass from replacing the service's public behavior. The interface remains the variation point: tests and applications can supply different repositories. This is often a more precise design than an open service hierarchy.

## How It Works

A final class cannot be extended. A final method cannot be overridden by a child. Since PHP 8.1, a class constant may be final; since PHP 8.4, a property may be final. Redeclaring any of these in an incompatible child is a compile-time error, before application code runs.

```php
class Policy
{
    final public const VERSION = 1;
    final protected string $name;

    final public function isAllowed(User $user): bool
    {
        return $user->isActive();
    }
}
```

The visibility of a final member still matters. A final protected method can be used by children but not replaced by them. A public final method is part of the externally visible invariant. A final property prevents a child from redeclaring the property; it does not by itself make the value read-only. Use `readonly` when the write lifecycle is the concern.

Private methods are not inherited in the normal overriding sense. As of PHP 8.0, private methods generally cannot be declared `final`, because final has no useful override restriction there; the constructor is the exception, which makes a private final constructor useful for controlled construction patterns.

## What PHP Does

PHP checks inheritance compatibility while loading class declarations. This means a forbidden extension or override fails before the request can call the affected method. It is a structural failure, not an exception that a normal `try`/`catch` around `new Child()` can reliably turn into a business result.

`final` is also independent of visibility and interfaces. A final class may implement an interface, and a final method may satisfy an interface method. Marking an implementation final does not make the interface final: another class may implement the interface differently.

## Minimal Example

```php
final class JsonResponse
{
    public function __construct(
        public readonly array $body,
        public readonly int $status = 200,
    ) {
    }
}

function respond(JsonResponse $response): string
{
    return json_encode($response->body, JSON_THROW_ON_ERROR);
}

echo respond(new JsonResponse(['ok' => true]));
```

The class is closed to subclassing because response construction and its data contract are intended to be stable. Formatting policy can still be extracted behind an interface if the application needs multiple formats.

## Practical Example: Closing an Invariant

Suppose a reservation must never be persisted before its interval has been validated. An open template method can be overridden incorrectly:

```php
abstract class UnsafeWorkflow
{
    public function run(Reservation $reservation): void
    {
        $this->validate($reservation);
        $this->persist($reservation);
    }

    protected function validate(Reservation $reservation): void {}
    abstract protected function persist(Reservation $reservation): void;
}
```

A child can override `run()` and skip validation. If the ordering is non-negotiable, close the algorithm and delegate only the intended variation:

```php
abstract class ReservationWorkflow
{
    final public function run(Reservation $reservation): void
    {
        $this->validate($reservation);
        $this->persist($reservation);
    }

    protected function validate(Reservation $reservation): void {}
    abstract protected function persist(Reservation $reservation): void;
}
```

This is the Template Method pattern with a deliberately narrow extension surface. Prefer composition when the variation is an ordinary dependency rather than a protected lifecycle hook.

## Bad Example and Better Example

An `OpenModel` base class with dozens of protected properties invites subclasses to couple themselves to representation. A `final` class with injected collaborators makes the same policy explicit:

```php
final class ReservationWriter
{
    public function __construct(private ReservationRepository $repository)
    {
    }

    public function write(Reservation $reservation): void
    {
        $this->repository->save($reservation);
    }
}
```

The better example is not “final everywhere.” If an actual stable variation is required, define an interface or a protected hook and test that contract. If consumers need to decorate behavior, use a collaborator or decorator instead of forcing them to inherit.

## Edge Cases

- `final` does not stop reflection, mutation of public mutable properties, or calls to public methods.
- A child can add new methods and properties unless the parent is final; it cannot replace a final member.
- A private constructor can support named constructors or a singleton-like controlled factory, but singleton state creates testing and lifecycle costs.
- Finalizing a method can be a source-compatibility break for downstream subclasses. Treat it as an API change in libraries.
- A final property and a readonly property answer different questions: “may the child redeclare this slot?” versus “may this object write this slot again?”

## Performance

`final` is primarily a correctness and API-contract tool. It may give an optimizer more information in some runtime situations, but it should not be added as a speculative micro-optimization. Profile real workloads; the cost of database calls, allocation, serialization, and I/O usually dominates dispatch details.

## Security

Closing a method can protect an authorization or validation step from being silently bypassed by a local subclass. It is not a security boundary against code that can modify the application. Security still depends on authentication, authorization, input validation, and database constraints. Make security-critical invariants final where inheritance is a realistic maintenance risk, then test the invariant at the boundary.

## Database Interaction

Finalizing `reserve()` cannot prevent a race between two PHP processes. Two requests can both pass an in-memory conflict check. The database must enforce the invariant with an appropriate unique constraint, exclusion strategy, transaction, or lock as discussed in later database chapters. `final` controls code structure; it does not serialize requests.

## Testing

Test the behavior that the final method protects, not the keyword itself:

```php
final class InMemoryReservationRepository implements ReservationRepository
{
    /** @var list<Reservation> */
    public array $saved = [];

    public function save(Reservation $reservation): void
    {
        $this->saved[] = $reservation;
    }
}

$repository = new InMemoryReservationRepository();
$service = new ReservationService($repository);
$service->reserve($reservation);

assert(count($repository->saved) === 1);
```

Add a regression test for every invariant: invalid intervals are rejected, persistence is not called after validation failure, and a repository failure is propagated or translated according to the application contract. Static analysis can also catch an attempted illegal override, but that code should never be part of a runnable test suite.

## Common Mistakes

- Marking every class final without deciding where extension or substitution is needed.
- Leaving a public mutable property open and assuming `final class` makes the object immutable.
- Using inheritance only to reuse implementation when composition would make dependencies visible.
- Believing final methods solve concurrency or database uniqueness.
- Adding final members to a public library without documenting the extension policy and migration impact.

## Senior Engineer Thinking

Ask: “What must never vary, and what should vary?” Close the algorithm that protects an invariant. Expose variation through a small interface or collaborator. The result is easier to test because the stable policy and replaceable dependency have different seams. A final class is a design statement about the public contract, not a badge of modern PHP style.

## Exercises

1. Convert a service with an overridable `run()` method into a final workflow with one injected strategy. List which behaviors remain variable.
2. Design a `final` value object for a reservation identifier. Decide whether its property should be public readonly or private with an accessor, and explain the trade-off.
3. Write a test proving that a repository is not called when interval validation fails. Then explain why the test does not prove database uniqueness.

## Review Questions

1. What does `final` restrict, and what does it not restrict?
2. Why might a final method plus an interface be preferable to an open base class?
3. How do final and readonly differ for a property?
4. Why can finalizing a library method be a compatibility break?
5. Why can a final reservation workflow still suffer a race condition?

## Summary

`final` closes inheritance-time variation. Use it to protect algorithms, invariants, properties, and constants that are not safe extension points. Keep real variation explicit through interfaces and composition, and remember that language-level closure does not replace database constraints, authorization, or tests.

## Official References

- [PHP Manual: Final Keyword](https://www.php.net/manual/en/language.oop5.final.php)
- [PHP Manual: Classes and Objects](https://www.php.net/manual/en/language.oop5.php)
