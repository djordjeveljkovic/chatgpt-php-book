---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 197
title: Domain-Driven Design
slug: domain-driven-design
status: complete
summary: ../../_ai/chapter-summaries/197-domain-driven-design-summary.md
---

# Chapter 197 — Domain-Driven Design

## Why This Matters

Domain-Driven Design (DDD) is a way to shape software around the business domain and the language experts use to describe it. It is most valuable when rules, workflows, state transitions, and terminology are difficult enough that a database schema or framework model no longer explains the system.

DDD does not mean every application needs aggregates, events, and a large class hierarchy. It begins with understanding the problem, drawing boundaries around different models, and making important rules explicit. A small application may use only a clear domain vocabulary and a few value objects; a larger one may need bounded contexts and explicit application services.

## Ubiquitous Language

A team needs shared words for concepts, actions, states, and policies. If product staff say “hold,” finance says “authorization,” and code says “pending,” the mismatch can hide a real difference in behavior. Record examples and forbidden ambiguities in a glossary, then use the chosen terms in requirements, tests, code, events, and support messages.

Language must be precise. “Customer” may mean the account holder in one context and the billed legal entity in another. “Cancelled” may mean a user requested cancellation or that a provider confirmed a refund. A shared language is discovered through conversations and examples, not invented by a developer in isolation.

## Bounded Contexts

A bounded context is a boundary within which a model and its terms have consistent meaning. A Reservation context can own court availability and booking transitions; a Billing context can own invoices and payment status. Both may refer to a customer but need not share one class or database schema.

Context boundaries reduce accidental coupling. Connect contexts through explicit APIs, events, or translation layers. An Anti-Corruption Layer protects one model from another model's terms and failure behavior. A shared kernel can work for a small, stable set of concepts, but it creates joint ownership and should remain deliberately small.

Do not split contexts merely by database table or team name. Find a boundary where rules, ownership, change cadence, and consistency needs differ. A context map should state who publishes facts, who owns decisions, and what happens when the other context is unavailable.

## Strategic and Tactical Design

Strategic design asks which domain capabilities matter, which are supporting, and which can use an external product. Tactical design gives a capability code structures such as entities, value objects, aggregates, repositories, domain services, and domain events.

Use the least elaborate structure that preserves the domain rule:

- a value object represents a validated concept with value equality;
- an entity has identity and a lifecycle;
- an aggregate defines a consistency boundary and root;
- a repository retrieves and stores an aggregate;
- a domain service holds a rule that does not belong naturally to one entity;
- an application service coordinates a use case and transaction;
- a domain event records a fact after a state transition.

These roles are boundaries, not mandatory folders. A class called Entity is not automatically a domain model.

## A Small Domain Model

Suppose a reservation can be confirmed only while it is held, and cancellation is allowed only before the start time. Put those transitions where the state is known:

~~~php
<?php

declare(strict_types=1);

enum ReservationStatus: string
{
    case Held = 'held';
    case Confirmed = 'confirmed';
    case Cancelled = 'cancelled';
}

final class Reservation
{
    public function __construct(
        private readonly string $id,
        private ReservationStatus $status,
        private readonly DateTimeImmutable $startsAt,
    ) {
    }

    public function confirm(): void
    {
        if ($this->status !== ReservationStatus::Held) {
            throw new DomainException('Only held reservations can be confirmed');
        }

        $this->status = ReservationStatus::Confirmed;
    }

    public function cancel(DateTimeImmutable $now): void
    {
        if ($now >= $this->startsAt || $this->status === ReservationStatus::Cancelled) {
            throw new DomainException('Reservation cannot be cancelled');
        }

        $this->status = ReservationStatus::Cancelled;
    }

    public function status(): ReservationStatus
    {
        return $this->status;
    }
}
~~~

The example still needs an identity, persistence mapping, authorization, and concurrency protection. DDD clarifies where the transition rule lives; it does not replace a database constraint or transaction.

## Application and Domain Boundaries

The domain model should not parse HTTP requests, read session globals, render JSON, or know PDO. An application service accepts a command, loads the relevant aggregate, invokes its behavior, persists the result, and publishes effects through an outbox or transaction boundary.

Infrastructure adapters translate external representations into domain values and map domain outcomes back to HTTP, messages, or CLI output. Keep mapping explicit. A provider's status code is not automatically a domain status, and an HTTP timeout is not automatically a business rejection.

## Modeling Workshops and Examples

Use event storming, example mapping, story conversations, or a simpler whiteboard session to discover commands, events, policies, actors, and exceptions. Ask:

- What can happen, and who is allowed to cause it?
- Which facts must be true before and after the action?
- Which terms are ambiguous?
- Which decisions need immediate consistency?
- Which failures can be retried?
- Which context owns the data and publishes the fact?

Examples are stronger than abstract nouns. “A held reservation expires after 15 minutes” reveals a clock, state transition, recovery job, and user-visible outcome. Turn the example into a test before choosing a class hierarchy.

## Persistence and Integration

Persist domain state in a form that can be queried and migrated. An ORM can map entities, but its identity map, lazy loading, callbacks, and flush behavior must not silently become the domain contract. A relational schema can be a good persistence model without being the domain model.

When a context publishes an event, define version, ordering, duplicate behavior, and retention. Consumers should not reach into another context's database to reconstruct its model. If a synchronous call is required, define timeout and fallback behavior; if eventual consistency is acceptable, make the lag visible.

## Failure and Evolution

Domain rules evolve. Keep public terms and event schemas versioned, preserve compatibility during migrations, and record decisions that would otherwise be lost in code. When a model boundary is wrong, introduce a translation layer and migrate gradually instead of sharing internals indefinitely.

DDD does not remove distributed failure. An aggregate cannot atomically update another context, a queue can deliver twice, and an external provider can time out after committing. Combine domain rules with idempotency, outbox delivery, authorization, and observability.

## Common Mistakes

- Treating DDD as a prescribed class and folder structure.
- Sharing one model across contexts with different meanings.
- Calling database rows entities without identity and behavior.
- Putting HTTP, SQL, or provider details into domain objects.
- Making every noun an aggregate and every state change an event.
- Ignoring authorization, concurrency, and delivery failure.
- Choosing abstractions before talking through concrete examples.

## Senior Engineer Thinking

DDD is disciplined modeling around a real domain and its change pressures. Establish useful language, draw boundaries where meaning and consistency differ, place invariants in the model that owns them, and keep application and infrastructure concerns at explicit edges. Use tactical patterns only when they make a rule or failure boundary easier to understand.

## Exercises

1. Map the language and contexts for a reservation, billing, and notification system. List terms that differ between contexts.
2. Turn three business examples into commands, invariants, events, and failure cases.
3. Identify a domain rule currently implemented in a controller and move it to the model that owns the required state.
4. Design an Anti-Corruption Layer between a legacy customer API and a new billing context.

## Review Questions

1. What problem does ubiquitous language solve?
2. How does a bounded context differ from a database schema?
3. Which tactical patterns are useful in a small domain, and which may be unnecessary?
4. Why should domain objects avoid HTTP and persistence details?
5. How do outbox and idempotency concerns remain relevant in DDD?

## Summary

Domain-Driven Design shapes software around a precise domain language, bounded contexts, owned invariants, and explicit application and infrastructure boundaries. Use entities, value objects, aggregates, repositories, services, and events when they clarify real rules and consistency needs. DDD complements transactions, authorization, idempotency, and observability; it does not replace them.

## References

- [Eric Evans: Domain-Driven Design](https://www.domainlanguage.com/ddd/)
- [Martin Fowler: Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- [Martin Fowler: Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [PHP enumerations](https://www.php.net/manual/en/language.enumerations.php)

