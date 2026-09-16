---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 32
title: Traits
slug: traits
status: complete
summary: ../../_ai/chapter-summaries/032-traits-summary.md
---

# Chapter 32 — Traits

## Why This Matters

A trait is a reusable group of methods, properties, and constants that a class can include with `use`. Traits address a limitation of single inheritance: unrelated classes may need the same small implementation without sharing an “is-a” relationship.

That convenience can hide dependencies. A trait’s method executes in the consuming class’s scope, can access its private members, and can collide with methods or properties supplied by another trait or the class itself. Treat a trait as source-level behavior composition with a contract, not as a miniature service that gets free access to the whole object.

## Mental Model

```php
trait RecordsAttempts
{
    /** @var list<string> */
    private array $attempts = [];

    private function recordAttempt(string $name): void
    {
        $this->attempts[] = $name;
    }
}

final class LoginService
{
    use RecordsAttempts;
}
```

The trait is not instantiated. Its members become members of `LoginService` as part of class composition. A trait can require that the consuming class provide methods with `abstract` declarations, which makes hidden assumptions visible to PHP and to readers.

## Core Concept

Traits are best for small, cohesive behavior that is genuinely orthogonal to the class’s main identity. A formatting helper with no state, a domain event recording capability, or a carefully scoped serialization helper may fit. A trait that reaches into a container, database, current user, and logger is a hidden inheritance hierarchy.

```php
<?php

declare(strict_types=1);

trait HasDomainEvents
{
    /** @var list<DomainEvent> */
    private array $events = [];

    protected function record(DomainEvent $event): void
    {
        $this->events[] = $event;
    }

    /** @return list<DomainEvent> */
    public function releaseEvents(): array
    {
        $events = $this->events;
        $this->events = [];

        return $events;
    }
}

final class Reservation
{
    use HasDomainEvents;

    public function confirm(): void
    {
        // Validate the state transition, then record the event.
        $this->record(new ReservationConfirmed());
    }
}
```

The trait owns the event buffer’s invariant and exposes a narrow API. The aggregate still owns when the event is recorded. The application must later decide how released events are persisted or published; a trait does not make delivery reliable.

## How It Works

The engine treats trait members as members available on the consuming class after composition. Method calls use the consuming object’s identity and scope. This explains both the usefulness and the danger: `$this` is the final class object, and a trait can accidentally depend on names it does not declare.

The exact compilation and method-table process is an implementation detail covered more fully in the under-the-hood volume. For application design, assume that a trait’s methods are part of the class API and that conflicts must be resolved when the class is declared.

## What PHP Does

When two traits define the same method, resolve the conflict explicitly:

```php
trait JsonLogging
{
    public function formatLog(array $context): string
    {
        return json_encode($context, JSON_THROW_ON_ERROR);
    }
}

trait TextLogging
{
    public function formatLog(array $context): string
    {
        return implode(' ', $context);
    }
}

final class Logger
{
    use JsonLogging, TextLogging {
        JsonLogging::formatLog insteadof TextLogging;
        TextLogging::formatLog as formatTextLog;
    }
}
```

`insteadof` chooses the primary implementation. `as` creates an alias; it does not rename the original away. Conflict resolution should be rare enough to understand at a glance. If every consumer needs a different precedence rule, the design probably wants explicit collaborators.

## Practical Example

A trait can express a local, reusable invariant with an explicit requirement:

```php
trait PublishesEvents
{
    abstract protected function eventPublisher(): EventPublisher;

    protected function publish(DomainEvent $event): void
    {
        $this->eventPublisher()->publish($event);
    }
}

final class ReservationNotifier
{
    use PublishesEvents;

    public function __construct(private EventPublisher $publisher) {}

    protected function eventPublisher(): EventPublisher
    {
        return $this->publisher;
    }
}
```

The abstract requirement makes the dependency explicit to PHP, but the trait still couples consumers to an `eventPublisher()` method name and lifecycle. For one or two classes, direct composition may be clearer.

## Production Example

Cross-cutting observability sometimes tempts teams to add a `Loggable` trait that knows a global logger and request ID. In a PHP-FPM request, a request ID may be available; in a CLI worker, it may need to be created; in a batch job, one operation may process many IDs. A trait can hide those differences and produce misleading logs. An injected logger or explicit decorator can make context and ownership visible, especially at process boundaries.

Traits are more defensible for framework-required small hooks or pure, domain-local behavior. Even then, document required properties and methods and keep the trait’s public surface small.

## Bad Example

```php
trait UsesEverything
{
    public function save(): void
    {
        $this->container->get(PDO::class)->exec($this->sql);
        $this->logger->info('saved');
        $this->cache->delete($this->key);
    }
}
```

This trait has undeclared dependencies, hidden transaction assumptions, and a name that says nothing about its behavior. Any class can appear to gain persistence while silently requiring three properties and a particular SQL field.

## Better Example

Prefer a collaborator for infrastructure behavior:

```php
final class ReservationWriter
{
    public function __construct(
        private ReservationRepository $repository,
        private EventPublisher $events,
    ) {}

    public function save(Reservation $reservation): void
    {
        $this->repository->add($reservation);
        $this->events->publish(new ReservationSaved($reservation->id()));
    }
}
```

The dependencies and ordering are visible, and a transaction or outbox can be introduced at the writer boundary. A trait would not provide those guarantees.

## Edge Cases

- A class method takes precedence over a trait method, and a trait method takes precedence over an inherited method; resolve design conflicts rather than relying on precedence accidentally.
- Trait properties become class properties. Two traits with incompatible properties of the same name can make class composition invalid.
- Private trait members are private members of the consuming class, so naming collisions can still be surprising.
- A trait can declare abstract methods, but that contract applies to every consumer.
- Trait static state can make tests and long-running workers unexpectedly share state. Prefer instance state or an explicit service.
- A trait cannot be used as a substitute for multiple inheritance of domain types; it supplies implementation, not substitutability.

## Performance

Traits do not create a separate runtime object or a network boundary. Their direct call cost is generally similar to an ordinary method on the consuming class. The costs to watch are duplicated state, repeated work, and the maintenance impact of behavior being copied into many classes. Fixing a trait may require checking all consumers.

## Security

Hidden trait dependencies can bypass security review. A trait that writes data, checks permissions, or handles tokens should declare what it needs and be tested in every relevant context. Do not use a trait to smuggle privileged clients or secrets into a class. Keep authorization policy explicit and near the resource boundary.

## Database Interaction

Do not put transaction boundaries in a generic trait. A transaction belongs to a unit of work whose owner understands all writes and side effects. A reusable trait may validate a value or collect domain events, but persisting those events and guaranteeing atomicity belongs to the repository/application boundary.

## Concurrency

Trait code runs in the same object and process as its consumer. It does not synchronize PHP-FPM workers or queue jobs. Static trait state is especially dangerous as a pseudo-lock or request cache in a long-running process. Use database constraints, atomic operations, or an explicit lock service for shared state.

## Testing

Test a trait through a representative consuming class:

```php
final class EventHolder
{
    use HasDomainEvents;

    public function addEvent(DomainEvent $event): void
    {
        $this->record($event);
    }
}

final class HasDomainEventsTest extends TestCase
{
    public function testEventsCanBeReleasedOnce(): void
    {
        $holder = new EventHolder();
        $event = new ReservationConfirmed();

        $holder->addEvent($event);

        self::assertSame([$event], $holder->releaseEvents());
        self::assertSame([], $holder->releaseEvents());
    }
}
```

If a trait is used by several materially different classes, test each integration context. A single convenient host class may not expose a missing property, altered visibility, or conflicting method in another consumer. Test behavior, not the fact that `use` appears in source.

## Common Mistakes

- Treating traits as free-floating services.
- Leaving required properties and methods undocumented.
- Using traits to avoid making a dependency explicit.
- Resolving many conflicts with `insteadof` instead of redesigning.
- Storing request or global state in static trait members.
- Forgetting that a trait change affects every consuming class.

## Senior Engineer Thinking

Ask whether the behavior is truly orthogonal, whether its dependencies can be declared, and whether consumers share the same invariant. If the answer is no, use an object collaborator or an interface. A good trait is small enough to explain in one paragraph, has a narrow surface, and does not make its host class depend on invisible infrastructure.

## Exercises

1. Write a trait that normalizes a value object without using undeclared properties. Decide whether a function would be clearer.
2. Create two conflicting traits and resolve them with `insteadof` and `as`. Explain the resulting public API.
3. Refactor a logging trait with hidden dependencies into an injected decorator.
4. Find static state in a trait and describe how it behaves differently in a web request and a long-running worker.

## Review Questions

1. How is a trait different from an abstract class and an interface?
2. Why can trait methods have dangerous hidden dependencies?
3. What do `insteadof` and `as` do?
4. Why is a trait not a concurrency mechanism?
5. When is composition a clearer replacement for a trait?

## Summary

Traits reuse implementation across otherwise unrelated classes, but their members become part of each consuming class. Keep traits cohesive, explicit about requirements, and small in public surface. Resolve conflicts deliberately, avoid hidden infrastructure and static state, and choose composition when behavior has meaningful dependencies or lifecycle.
