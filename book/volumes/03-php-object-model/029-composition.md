---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 29
title: Composition
slug: composition
status: complete
summary: ../../_ai/chapter-summaries/029-composition-summary.md
---

# Chapter 29 — Composition

## Why This Matters

Composition builds an object from other objects. A reservation service can use a clock, repository, conflict checker, and event publisher without inheriting their implementation. This keeps responsibilities local and makes a dependency visible at the point where the service is created.

Composition is not automatically simple. A graph with twenty tiny services can be harder to understand than one class with a clear algorithm. The engineering question is which parts vary independently, have separate failure modes, or deserve separate tests and ownership.

## Mental Model

Inheritance says “this object can be used as that object.” Composition says “this object delegates part of its work to those collaborators.” The composed object controls the collaboration protocol: when a dependency is called, what data crosses the boundary, and how failures are handled.

```text
ReservationService
 ├── ReservationRepository
 ├── ConflictChecker
 ├── Clock
 └── EventPublisher
```

The graph can be assembled with constructor injection. The object owns the relationship, but it does not necessarily own the lifetime of every collaborator. A service may use a shared database connection without closing it; the application container that created the connection owns its lifecycle.

## Core Concept

Start with the rule, then identify collaborators:

```php
<?php

declare(strict_types=1);

final class ReservationService
{
    public function __construct(
        private ReservationRepository $reservations,
        private ConflictChecker $conflicts,
        private Clock $clock,
    ) {}

    public function create(UserId $user, CourtId $court, TimeRange $range): Reservation
    {
        if ($range->startsBefore($this->clock->now())) {
            throw new DomainException('A reservation cannot start in the past');
        }

        if ($this->conflicts->exists($court, $range)) {
            throw new DomainException('The court is already reserved');
        }

        $reservation = Reservation::make($user, $court, $range);
        $this->reservations->add($reservation);

        return $reservation;
    }
}
```

The service orchestrates. It does not need to know whether the repository uses PDO, an HTTP API, or an in-memory fake. The conflict checker can start with an O(n) scan and later move the query to the database without changing the business-facing service contract.

## How It Works

Each collaborator is an object handle. Calling a method traverses the graph and creates another call frame in the current PHP process. The engine does not make composition asynchronous or isolate failures: a thrown exception still propagates through the call stack unless a boundary catches it.

This is why a dependency boundary needs a failure policy. A repository timeout is not the same as “court is occupied.” Map infrastructure failure to an operational error and domain conflict to a domain outcome; do not turn every exception into “unavailable.”

## What PHP Does

Constructor property promotion and `readonly` can express stable dependencies:

```php
final class BillingService
{
    public function __construct(
        private readonly PaymentGateway $gateway,
        private readonly ReceiptStore $receipts,
    ) {}
}
```

`readonly` prevents reassignment of the property after initialization. It does not make the gateway object immutable, and it does not make calls safe to repeat. A composed design still needs explicit state and retry semantics.

## Practical Example

Use a small value object and a policy object rather than subclassing every service for a different cutoff:

```php
final class BusinessHoursPolicy
{
    public function allows(TimeRange $range): bool
    {
        return (int) $range->startsAt()->format('N') <= 5
            && (int) $range->startsAt()->format('G') >= 8
            && (int) $range->endsAt()->format('G') <= 18;
    }
}

final class ReservationValidator
{
    public function __construct(private BusinessHoursPolicy $hours) {}

    public function validate(TimeRange $range): void
    {
        if (!$this->hours->allows($range)) {
            throw new DomainException('Outside business hours');
        }
    }
}
```

A holiday policy, venue-specific schedule, or test policy can be composed later. The variation is explicit instead of encoded in a growing inheritance tree.

## Production Example

Suppose a reservation write follows this sequence:

```text
check availability → insert reservation → publish confirmation
```

Composition makes the boundaries visible, but it does not solve the race between two requests. Both requests can pass the check before either inserts. The database transaction and constraint must enforce the invariant; the composed service should translate a uniqueness or conflict failure into a safe domain response. Publishing a confirmation may belong in an outbox transaction rather than a direct network call after the insert.

This is the difference between decomposing code and designing a reliable system. Collaborators clarify responsibility; transactions, indexes, and queues clarify consistency and failure.

## Bad Example

```php
final class ReservationService
{
    public function create(array $input): void
    {
        $pdo = new PDO($_ENV['DB_DSN']);
        $mailer = new SmtpMailer($_ENV['SMTP_HOST']);
        $now = new DateTimeImmutable();

        // Validation, SQL, email, and response formatting all happen here.
    }
}
```

This is composition hidden inside a method, with construction, configuration, time, I/O, and policy coupled together. Tests need live infrastructure and cannot control time or failure independently.

## Better Example

Build the graph at the application boundary:

```php
$clock = new SystemClock();
$repository = new PdoReservationRepository($pdo);
$checker = new SqlConflictChecker($pdo);
$service = new ReservationService($repository, $checker, $clock);
```

The domain-facing class remains deterministic when given a fake clock, fake repository, and controlled checker. The composition root is allowed to know concrete classes; the core service need not.

## Edge Cases

- A collaborator can be optional, but nullable dependencies often hide two modes. Prefer two explicit policies when behavior genuinely differs.
- Do not inject a service locator and call arbitrary services from inside a class; it hides the graph and weakens static analysis.
- Avoid circular dependencies. They often indicate that a workflow should be moved to a third coordinator or that responsibilities are mixed.
- Decide ownership for resources. The object that opens a file or transaction should normally close or complete it.
- A decorator composes an object with the same interface to add logging, caching, authorization, or metrics without subclassing.

## Performance

An extra in-process method call is usually negligible next to a database or network round trip. The real costs are duplicated work, repeated queries, serialization between boundaries, and excessive allocation. Composition can improve performance when it allows caching or batching, but it can also create an N+1 call graph if each small collaborator queries independently. Trace calls and inspect query counts rather than guessing.

## Security

Dependency injection is not authorization. A caller who can construct a service with a permissive policy may bypass a boundary if construction is exposed carelessly. Keep security checks at the correct application boundary, pass least-privilege clients, and avoid composing secrets into objects that do not need them.

## Database Interaction

Keep selection and conflict enforcement near the database. A PHP conflict checker that loads every reservation has O(n) application memory and creates a race unless the write is protected. Prefer an indexed query for candidate conflicts and a transaction or database constraint for the final invariant. The service can still own the domain interpretation of the database result.

## Concurrency

Each PHP-FPM request may have its own object graph, but requests share database state. Two separately composed services do not coordinate merely because they contain the same class. Use transactions, unique constraints, locks, or idempotency keys at the shared-state boundary. For asynchronous events, make publication retryable and observable.

## Testing

Composition enables focused tests:

```php
final class ReservationServiceTest extends TestCase
{
    public function testPastReservationsAreRejectedBeforePersistence(): void
    {
        $repository = new InMemoryReservationRepository();
        $service = new ReservationService(
            $repository,
            new AlwaysAvailableChecker(),
            new FrozenClock(new DateTimeImmutable('2026-01-01 12:00:00 UTC')),
        );

        $this->expectException(DomainException::class);
        $service->create(
            UserId::fromString('user-1'),
            CourtId::fromString('court-1'),
            TimeRange::fromStrings('2025-12-31 10:00:00 UTC', '2025-12-31 11:00:00 UTC'),
        );
    }
}
```

Use fakes for business-rule tests and integration tests for PDO queries, transaction behavior, and real constraints. A fake can prove orchestration while an integration test proves the database actually enforces it.

## Common Mistakes

- Creating dependencies inside business methods.
- Splitting every line into a class without independent responsibility.
- Assuming composition removes the need for transactions.
- Passing arrays across every boundary, losing names and invariants.
- Mocking every collaborator so that no real behavior remains under test.

## Senior Engineer Thinking

Draw the dependency graph and mark each edge as in-process, database, network, or asynchronous. For each edge, name its owner, latency, failure mode, and test boundary. Compose around independent reasons to change, not around a slogan such as “single responsibility.” Keep a small graph when the system is small, and let operational boundaries—not fashion—justify more infrastructure.

## Exercises

1. Refactor a class that constructs its repository and clock internally so both are injected.
2. Design a reservation composition graph and label the component that owns each transaction and resource.
3. Add a caching decorator to a repository. Decide how stale data affects availability and write a test for the policy.
4. Count database queries in a composed search operation. Find and remove an N+1 call pattern.

## Review Questions

1. What relationship does composition express compared with inheritance?
2. Why does composition not by itself solve a reservation race?
3. What is a composition root?
4. When is a small class graph worse than a larger class?
5. Which failures must be handled at a database or network boundary rather than hidden in a collaborator?

## Summary

Composition assembles behavior from collaborators and makes variation, ownership, and tests explicit. It avoids many inheritance couplings, but it can still become accidental complexity. Inject stable dependencies, keep the graph purposeful, and design transactions, failures, concurrency, and security at the boundaries where they actually exist.
