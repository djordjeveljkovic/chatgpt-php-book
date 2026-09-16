---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 28
title: Inheritance
slug: inheritance
status: complete
summary: ../../_ai/chapter-summaries/028-inheritance-summary.md
---

# Chapter 28 — Inheritance

## Why This Matters

Inheritance lets one class extend another. It can model a genuine “is-a” relationship and provide substitutability: code written for the parent contract can accept a child. It can also create a rigid hierarchy in which a small change in a base class affects many unrelated features.

Experienced PHP design treats inheritance as a semantic promise, not a shortcut for sharing code. If a subclass cannot honor the parent’s behavior, the hierarchy is lying to its callers. In many application services, composition gives the same reuse with fewer hidden couplings.

## Mental Model

With `extends`, a child class receives the accessible behavior of one parent class and may add or override behavior. PHP supports single class inheritance: a class has one direct parent, though it may implement many interfaces and use many traits.

The child is still a valid instance of its parent type. That relationship should remain true for every public operation. A `Square` that breaks callers expecting an independently settable `Rectangle` is a classic violation: the names look related, but the behavior is not substitutable.

## Core Concept

```php
<?php

declare(strict_types=1);

class Report
{
    public function format(): string
    {
        return 'report';
    }
}

final class HtmlReport extends Report
{
    public function format(): string
    {
        return '<p>report</p>';
    }
}

function render(Report $report): string
{
    return $report->format();
}
```

`render(new HtmlReport())` works because the child can be used where `Report` is expected. The `final` modifier here prevents further extension of this leaf class; it is a design choice, not a requirement for inheritance.

Visibility and signatures constrain overrides. A child cannot make an inherited public method less visible. Parameter types may be widened and return types may be narrowed according to PHP’s variance rules, but changing behavior still can violate the semantic contract even when the type checker accepts the code.

## How It Works

The class declaration records a parent relationship and method tables. A method call made through a parent-typed variable can dispatch to an overridden child method. This is dynamic dispatch, not a copy of source text at the call site. Private parent members remain private to the parent; a child property with the same name is a separate property, not an override of the parent’s private state.

The engine details are implementation-specific, so the useful engineering model is simpler: the runtime resolves the actual object’s method implementation while checking that the call is valid for the declared type. `parent::method()` explicitly selects the parent implementation and is not the same as dynamic dispatch through `$this`.

## What PHP Does

Constructors deserve special care. If a child declares its own constructor, PHP does not automatically call the parent constructor:

```php
class ConnectionClient
{
    public function __construct(protected string $endpoint) {}
}

final class ApiClient extends ConnectionClient
{
    public function __construct(string $endpoint, private int $timeoutSeconds)
    {
        parent::__construct($endpoint);
    }
}
```

Omitting `parent::__construct()` can leave parent invariants uninitialized. A child must know what it is promising before it decides whether and how to initialize the base class.

## Practical Example

Inheritance is appropriate when variants share a stable domain contract and the parent owns meaningful common policy:

```php
abstract class Exporter
{
    final public function export(array $rows): string
    {
        $this->validate($rows);

        return $this->encode($rows);
    }

    protected function validate(array $rows): void
    {
        if ($rows === []) {
            throw new InvalidArgumentException('At least one row is required');
        }
    }

    abstract protected function encode(array $rows): string;
}

final class JsonExporter extends Exporter
{
    protected function encode(array $rows): string
    {
        return json_encode($rows, JSON_THROW_ON_ERROR);
    }
}
```

The template method fixes validation and delegates only representation. A caller cannot bypass validation by overriding `export` because it is final. This is a deliberate narrow extension point.

## Production Example

An application may have `CardPayment` and `BankTransferPayment` classes. If every payment must expose `authorize`, `capture`, and `refund` with the same guarantees, a shared parent can centralize common state transitions. But if card payments need asynchronous 3-D Secure flows while transfers have settlement windows, forcing both into one lifecycle may create conditionals and unsafe “not applicable” methods. An interface for the capability plus composed workflow objects may represent the domain more honestly; Chapter 30 develops that choice.

## Bad Example

```php
class BaseController
{
    protected function loadEverything(): array { /* ... */ return []; }
}

final class ReservationController extends BaseController
{
    // Inherits database access, authentication, formatting, caching, and logging by accident.
}
```

This base class is a grab bag. Its children are coupled to protected methods and undocumented lifecycle assumptions. A change to one concern can break every controller.

## Better Example

Compose focused dependencies at the boundary:

```php
final class ReservationController
{
    public function __construct(
        private ReservationService $service,
        private ReservationPresenter $presenter,
    ) {}

    public function create(CreateReservationRequest $request): Response
    {
        $reservation = $this->service->create($request->validated());

        return $this->presenter->response($reservation);
    }
}
```

The controller’s behavior is visible in its constructor, and each dependency can be tested independently. Inheritance is still useful for a stable framework hook or a true specialization, but shared convenience is not enough by itself.

## Edge Cases

- A class can extend only one class but can implement multiple interfaces.
- A child cannot override a `final` method or extend a `final` class.
- Static methods are inherited, but calls through a class name and calls through `static::` have different late-static-binding behavior.
- Parent private properties and methods cannot be accessed directly by a child.
- Overriding a method with an incompatible signature is a declaration error; compatible typing does not guarantee compatible semantics.
- Calling `parent::method()` from an override is useful for an invariant, but repeated parent calls in a deep hierarchy become hard to reason about.

## Performance

Inheritance usually is not selected for raw speed. Method dispatch and object allocation are small compared with network and database work, while hierarchy complexity can impose a larger maintenance cost. Avoid speculative base classes; measure hot paths if dispatch is actually significant.

## Security

Do not make security-critical hooks `protected` merely so subclasses can replace them. A subclass supplied by a package or future maintainer can weaken an assumption in the base class. Keep authorization policy explicit, minimize extension points, and make security invariants final or enforce them at a boundary that subclasses cannot bypass.

## Testing

Test substitutability at the parent contract:

```php
final class ExporterTest extends TestCase
{
    public function testJsonExporterHonorsTheExporterWorkflow(): void
    {
        $exporter = new JsonExporter();

        self::assertSame('[{"id":1}]', $exporter->export([['id' => 1]]));
    }
}
```

For each subclass, test the behavior promised by the parent plus its specialized behavior. A test that only checks inherited methods execute may miss a violation such as silently dropping data, changing exception meaning, or accepting an invalid state.

## Common Mistakes

- Extending a concrete class solely to reuse a helper method.
- Treating `protected` state as a safe public extension API.
- Forgetting parent construction.
- Adding a child-only condition to a parent method until the base class knows every subtype.
- Using inheritance where the relationship is “has a” or “uses a”.
- Making a hierarchy deep enough that the effective behavior requires reading five classes.

## Senior Engineer Thinking

Before writing `extends`, write the parent promise in ordinary language. Can every child honor it without surprising callers? Is the shared behavior stable, or merely duplicated today? Would a small interface and composed collaborators isolate change better? A hierarchy is valuable when it expresses a durable substitution relationship; otherwise it is a dependency graph disguised as ancestry.

## Exercises

1. Create a `Notifier` parent contract and two subclasses. List the behaviors each subclass must preserve.
2. Repair a child constructor that forgets `parent::__construct()` and add a test for the parent invariant.
3. Find a base class in a codebase and classify each protected method as stable extension point, accidental helper, or policy that should be final.
4. Explain why mutable `Rectangle`/`Square` inheritance can violate substitutability.

## Review Questions

1. What does a child promise when it extends a parent?
2. When does PHP call the parent constructor automatically?
3. How do `parent::method()` and `$this->method()` differ?
4. Why can compatible method types still produce an invalid subclass?
5. What evidence would make inheritance preferable to composition?

## Summary

Inheritance models substitution, not merely code reuse. PHP supports one parent class, dynamic dispatch, controlled visibility, variance, and explicit parent calls. Use a narrow, stable parent contract, protect invariants, and prefer composition when the relationship is not genuinely “is-a.”
