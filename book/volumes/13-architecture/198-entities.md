---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 198
title: Entities
slug: entities
status: complete
summary: ../../_ai/chapter-summaries/198-entities-summary.md
---

# Chapter 198 — Entities

## Why This Matters

An entity is a domain object whose identity remains meaningful as its attributes change. Two reservations may have equal status and times but still be different reservations; their identity determines which one can be cancelled, paid, or audited.

An entity is more than a database row with an ID. It owns behavior and invariants that depend on its identity and lifecycle. Keep persistence identifiers, domain identity, authorization, and concurrency rules explicit rather than exposing a mutable bag of fields.

## Identity and Equality

Entity equality is based on a stable identity within a defined context. A public integer ID, UUID, or other identifier can work if its scope and generation rules are clear. Do not compare entities by every property: status and timestamps change, while identity does not.

Identity scope matters. An ID unique only within a tenant must be compared with tenant context, and an external provider ID should not be confused with the domain entity ID. Value objects used as identifiers can make format and scope explicit.

~~~php
<?php

declare(strict_types=1);

final readonly class ReservationId
{
    public function __construct(public string $value)
    {
        if (!preg_match('/^[a-z0-9_-]{8,64}$/D', $value)) {
            throw new InvalidArgumentException('Invalid reservation identifier');
        }
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }
}

final class Reservation
{
    public function __construct(
        private readonly ReservationId $id,
        private ReservationStatus $status,
    ) {
    }

    public function id(): ReservationId
    {
        return $this->id;
    }

    public function sameIdentityAs(self $other): bool
    {
        return $this->id->equals($other->id);
    }
}
~~~

ReservationStatus is assumed to be the enum from the DDD example. In a complete namespace, import or define it in the domain module. The identity method is intentionally explicit so callers do not accidentally use object reference equality.

## Protect Invariants Through Methods

A mutable entity should expose commands that preserve valid state instead of public setters. A status transition can check the current state, authorization-relevant data, and time before changing fields.

~~~php
<?php

declare(strict_types=1);

final class Account
{
    private bool $enabled = true;

    public function __construct(private readonly int $id)
    {
    }

    public function disable(string $reason): void
    {
        if (trim($reason) === '') {
            throw new InvalidArgumentException('A reason is required');
        }

        $this->enabled = false;
    }

    public function canAuthenticate(): bool
    {
        return $this->enabled;
    }
}
~~~

The entity can enforce local invariants. A rule requiring another aggregate, a database uniqueness check, or an external provider belongs in a domain service, aggregate boundary, or application transaction. Do not put every rule in one entity simply because it is convenient.

## Lifecycle and Creation

An entity can be new, persisted, archived, or deleted according to the domain. Decide when identity is assigned. Assigning a UUID before persistence makes references and outbox records straightforward; database-generated IDs can work when the persistence boundary owns identity creation.

Use named constructors for meaningful creation states:

~~~php
<?php

declare(strict_types=1);

final class Subscription
{
    private function __construct(
        private readonly string $id,
        private string $status,
    ) {
    }

    public static function start(string $id): self
    {
        if ($id === '') {
            throw new InvalidArgumentException('Subscription ID is required');
        }

        return new self($id, 'active');
    }

    public static function reconstitute(string $id, string $status): self
    {
        if (!in_array($status, ['active', 'paused', 'ended'], true)) {
            throw new InvalidArgumentException('Unknown subscription status');
        }

        return new self($id, $status);
    }
}
~~~

Separating creation from reconstitution prevents persistence data from being mistaken for a new business action. Reconstitution should validate enough to protect the object while allowing historical states that the current creation path no longer permits; migration policy determines whether old states are repaired or rejected.

## Persistence and Encapsulation

A mapper or repository may need to inspect state without making every field publicly writable. Use explicit hydration methods, a private constructor, or a persistence adapter that maps known fields. Avoid reflection or ORM magic becoming the only explanation of an entity's lifecycle.

Do not serialize an entity directly as an API response. Public representations are compatibility contracts and often omit internal state. Map explicitly to a response DTO, and authorize the resource before exposing it.

Optimistic version fields can detect concurrent changes:

~~~php
<?php

declare(strict_types=1);

final class VersionedDocument
{
    public function __construct(
        private readonly int $id,
        private int $version,
        private string $title,
    ) {
    }

    public function rename(string $title): void
    {
        if (trim($title) === '') {
            throw new InvalidArgumentException('Title is required');
        }

        $this->title = $title;
    }

    public function version(): int
    {
        return $this->version;
    }

    public function title(): string
    {
        return $this->title;
    }
}
~~~

The repository must include version in an atomic update and increment it only after a successful write. A version property alone does not prevent lost updates.

## Entities and Aggregate Boundaries

An entity inside an aggregate is governed by the aggregate root. Callers should not load a child from a repository and mutate it around the root's invariants. The root exposes operations that coordinate changes and emits domain events after valid transitions.

Not every domain object needs a persistent identity. Use a value object for concepts whose equality is entirely based on value and that have no independent lifecycle. A date range, email address, or money amount should not become an entity merely because it is stored in a column.

## Testing and Operations

Test entity behavior through public commands: valid transitions, invalid transitions, equality, creation, reconstitution, and invariant failures. Avoid tests that assert private fields or constructor implementation. Test persistence mapping against the real database and test concurrency using version or lock behavior at the repository boundary.

Record identity and state transitions in audit events without exposing secrets. An entity's identity should be stable across logs, messages, and support tools, but tenant scope and privacy policy still apply. Do not leak internal identifiers where a public opaque reference is required.

## Common Mistakes

- Comparing entities by all fields instead of identity.
- Exposing public setters that bypass transitions.
- Treating ORM hydration as domain creation.
- Generating identity with predictable values.
- Putting cross-aggregate and external rules into one entity.
- Assuming a version property prevents lost updates without an atomic write.
- Returning entities directly from an API.

## Senior Engineer Thinking

An entity gives a domain concept identity, lifecycle, and behavior. Define equality and identity scope, enforce local invariants through commands, separate creation from reconstitution, keep persistence and API mapping explicit, and coordinate cross-entity rules through aggregate and application boundaries.

## Exercises

1. Define entity identity and equality for a multi-tenant reservation system.
2. Replace public setters on an account or order with valid state-transition methods.
3. Add optimistic version handling to a repository update and test two concurrent edits.
4. Decide whether three domain concepts are entities or value objects, and justify the lifecycle difference.

## Review Questions

1. What makes an object an entity rather than a value object?
2. Why should equality use stable identity?
3. How should persistence reconstitution differ from business creation?
4. Which invariants belong outside one entity?
5. Why does a version property require an atomic repository update?

## Summary

Entities represent domain concepts with stable identity, lifecycle, and behavior. Define identity scope and equality, expose valid commands instead of unrestricted setters, separate creation from persistence reconstitution, map entities explicitly at API and database boundaries, and use aggregate and repository contracts for cross-entity invariants and concurrency.

## References

- [Martin Fowler: Entity](https://martinfowler.com/eaaCatalog/identityField.html)
- [Eric Evans: Domain-Driven Design](https://www.domainlanguage.com/ddd/)
- [PHP readonly properties](https://www.php.net/manual/en/language.oop5.basic.php)
- [PHP UUID extension documentation](https://www.php.net/manual/en/book.uuid.php)

