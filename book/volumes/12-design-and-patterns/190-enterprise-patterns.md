---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 190
title: Enterprise Patterns
slug: enterprise-patterns
status: complete
summary: ../../_ai/chapter-summaries/190-enterprise-patterns-summary.md
---

# Chapter 190 — Enterprise Patterns

## Why This Matters

Enterprise patterns describe recurring problems in systems with business rules, persistence, transactions, external integrations, and multiple entry points. They help a team name a boundary and reason about consistency. They do not turn a CRUD application into a better system merely by adding layers.

Choose a pattern from a concrete pressure: a transaction spans several writes, a domain invariant must survive multiple callers, a database model differs from the domain model, or a read path has different scaling needs from a write path. The pattern must state what it guarantees and what it leaves to the database, queue, or operator.

## Transaction Script and Domain Model

A Transaction Script organizes one operation as a procedure: load data, validate, write changes, and publish effects. It can be clear for a small workflow with simple rules.

~~~php
<?php

declare(strict_types=1);

function cancelReservation(PDO $db, int $reservationId, int $userId): void
{
    $db->beginTransaction();
    try {
        $query = $db->prepare(
            'SELECT status FROM reservations WHERE id = ? AND user_id = ? FOR UPDATE',
        );
        $query->execute([$reservationId, $userId]);
        $reservation = $query->fetch(PDO::FETCH_ASSOC);

        if ($reservation === false || $reservation['status'] === 'cancelled') {
            throw new DomainException('Reservation cannot be cancelled');
        }

        $update = $db->prepare(
            'UPDATE reservations SET status = ? WHERE id = ?',
        );
        $update->execute(['cancelled', $reservationId]);
        $db->commit();
    } catch (Throwable $exception) {
        $db->rollBack();
        throw $exception;
    }
}
~~~

A script must still define authorization, lock behavior, isolation, and event publication. It is not automatically unsafe because it is procedural.

A Domain Model puts behavior and invariants on domain objects and services. It can express complex rules across states and aggregates, but it introduces object mapping and lifecycle decisions. Use it when the business rules are rich enough that procedural scripts repeat or obscure them. Do not create an anemic class that only holds fields while every rule remains scattered in handlers.

## Service Layer

A Service Layer defines the application operations available to controllers, commands, jobs, and other entry points. It coordinates authorization, transactions, domain operations, and outgoing effects without becoming a second domain model.

~~~php
<?php

declare(strict_types=1);

interface Transaction
{
    /** @template T */
    /** @param callable(): T $operation */
    public function run(callable $operation): mixed;
}

final class ReservationApplicationService
{
    public function __construct(
        private ReservationRepository $reservations,
        private Transaction $transaction,
        private ReservationNotifier $notifier,
    ) {
    }

    public function cancel(int $actorId, int $reservationId): void
    {
        $this->transaction->run(function () use ($actorId, $reservationId): void {
            $reservation = $this->reservations->forUpdate($reservationId);
            if ($reservation === null || !$reservation->canBeCancelledBy($actorId)) {
                throw new DomainException('Reservation cannot be cancelled');
            }

            $reservation->cancel();
            $this->reservations->save($reservation);
            $this->notifier->queueCancelled($reservation);
        });
    }
}
~~~

The example assumes the notification is an outbox-backed effect or another mechanism that cannot be lost between the database commit and delivery. A service layer should not make a remote HTTP call while holding a database transaction open unless the consistency protocol explicitly requires it.

## Repository and Data Mapper

A Repository presents collection-like access to domain objects while hiding persistence queries. A Data Mapper translates between domain objects and rows without requiring the objects to know SQL or ORM lifecycle details.

~~~php
<?php

declare(strict_types=1);

final readonly class Reservation
{
    public function __construct(
        public int $id,
        public int $userId,
        public string $status,
    ) {
    }

    public function canBeCancelledBy(int $actorId): bool
    {
        return $this->userId === $actorId && $this->status === 'confirmed';
    }

    public function cancel(): self
    {
        return new self($this->id, $this->userId, 'cancelled');
    }
}

interface ReservationRepository
{
    public function forUpdate(int $id): ?Reservation;
    public function save(Reservation $reservation): void;
}
~~~

The repository contract should describe behavior the application needs, such as whether forUpdate locks a row and how a missing object is represented. Avoid a generic repository with methods for every SQL operation; it can hide query cost and allow invalid aggregate updates.

A Data Mapper owns conversion and persistence details. It should map explicit fields, handle optimistic versions or locking, and preserve null and enum semantics. Test it against the real database engine; an in-memory array does not reproduce indexes, transactions, isolation, or constraint errors.

## Unit of Work and Identity Map

A Unit of Work tracks changes during one application operation and writes them as a coordinated set. An Identity Map ensures that one database identity corresponds to one in-memory object within a scope, avoiding contradictory copies and duplicate writes. These patterns can simplify aggregate persistence but add lifecycle and memory complexity.

In a PHP-FPM request, the scope is usually one request. In a long-running worker, clear the unit after every job; retaining objects can leak tenant data and stale state across messages. A Unit of Work is not a substitute for a database transaction. The database still provides atomicity and conflict behavior.

Use explicit flush boundaries and inspect generated queries. Automatic dirty tracking can issue unexpected writes, load large graphs, or make a read operation expensive. If the domain has a small number of writes, explicit repository operations may be easier to reason about.

## Table Module and Active Record

A Table Module organizes operations around a database table or record set. It can be effective for straightforward reporting and CRUD workflows where the table is the dominant model. An Active Record object combines row data and persistence methods, which can be productive for simple applications.

Both patterns become awkward when one business operation spans several aggregates, external effects, or different persistence models. Do not let a convenient save method imply that a multi-step operation is atomic. Make transaction scope and authorization visible at the application boundary.

## CQRS and Read Models

Command Query Responsibility Segregation separates models or paths for changing state and reading it. It is justified when read and write concerns have materially different performance, shape, authorization, or scaling requirements. It can be as small as separate query handlers and command handlers over one database; it does not require event sourcing or a message broker.

A read model may be denormalized or asynchronously updated. Document freshness, rebuild strategy, and authorization data. A dashboard that is eventually consistent should show a timestamp or state that helps users understand the delay. Never use a stale read model for a permission decision or financial invariant without a defined consistency check.

## Event Sourcing and Outbox

Event Sourcing stores an ordered history of domain events as the source of state and rebuilds an aggregate by replaying them. It can provide auditability and temporal reconstruction, but event schemas, replay cost, correction policy, and privacy retention become core design concerns. A normal entity table plus an audit log may be simpler when full event reconstruction is not required.

An outbox records an event in the same transaction as the state change and publishes it asynchronously. It solves the crash window between database commit and message publication. It does not guarantee that consumers process the event exactly once; delivery IDs and idempotent consumers remain necessary.

## Choosing a Pattern

Evaluate a pattern against the problem:

1. What consistency or change pressure exists?
2. Which component owns the invariant?
3. What is the transaction and failure boundary?
4. What data does the operation need, and at what cost?
5. How will retries, concurrency, and authorization work?
6. How will the state be observed, rebuilt, migrated, and operated?
7. What simpler design was considered?

Write the contract before the class diagram. A Repository that promises a locked row, a CQRS query that is eventually consistent, and an outbox that retries delivery have different operational meanings. Name those meanings in documentation and tests.

## Testing and Operations

Test domain invariants with focused unit tests, repository and mapper behavior against the real database, service transactions and authorization through application tests, and message delivery with integration and contract tests. Test rollback, duplicate commands, deadlocks, stale reads, rebuilds, and partial provider failures where the pattern introduces them.

Instrument transaction duration, query count, outbox age, read-model lag, unit-of-work size, replay failures, and cache or identity-map hits. Log correlation and aggregate identifiers with privacy controls. A pattern that hides cost is a production risk; measure at the boundary where it matters.

## Common Mistakes

- Adding a repository, service, and unit of work to every CRUD screen.
- Assuming a domain object makes a database update atomic.
- Calling a remote provider inside an open transaction without a consistency plan.
- Treating an in-memory fake as proof of database behavior.
- Using stale read models for authorization or financial decisions.
- Choosing event sourcing without a replay, schema, and retention plan.
- Calling an outbox exactly-once delivery mechanism.
- Letting long-running workers retain scoped objects between jobs.

## Senior Engineer Thinking

Enterprise patterns are names for consistency, ownership, and change boundaries. Select the smallest pattern that addresses a measured pressure, define its transaction and failure semantics, and test the real infrastructure where fakes cannot model it. Keep read models, event histories, repositories, and services observable and reversible.

## Exercises

1. Compare a Transaction Script and Domain Model for a reservation cancellation workflow with two state invariants.
2. Design a Repository contract that states locking, missing-row, version, and transaction behavior.
3. Add an outbox to a service and list every crash point between commit, publish, and consumer handling.
4. Decide whether a reporting dashboard needs CQRS, a query handler over existing tables, or neither.
5. Define the cleanup boundary for a Unit of Work in a long-running PHP worker.

## Review Questions

1. When is a Transaction Script a reasonable choice?
2. What should a Service Layer coordinate?
3. Which guarantees belong in a Repository contract?
4. Why is a Unit of Work not a database transaction?
5. What does CQRS add, and what complexity does it introduce?
6. What problem does an outbox solve, and what does it not solve?
7. Which read-model data must never be trusted for authorization?

## Summary

Enterprise patterns organize application, domain, persistence, and integration boundaries around real consistency and change pressures. Transaction Scripts, Domain Models, Service Layers, Repositories, Mappers, Units of Work, CQRS, and outboxes each carry explicit costs and guarantees. Choose the smallest useful pattern, define transaction and failure semantics, test real infrastructure, and observe the resulting queries, lag, retries, and memory.

## References

- [Martin Fowler: Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
- [Martin Fowler: Repository](https://martinfowler.com/eaaCatalog/repository.html)
- [Martin Fowler: Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html)
- [Martin Fowler: CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Martin Fowler: Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [PHP PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)

