---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 181
title: SOLID
slug: solid
status: complete
summary: ../../_ai/chapter-summaries/181-solid-summary.md
---

# Chapter 181 — SOLID

## Why This Matters

SOLID is a set of design heuristics for managing change in object-oriented code. The principles are useful when a class has multiple reasons to change, a caller depends on more than it needs, or replacing an implementation changes behavior unexpectedly. They are not a requirement to split every function into interfaces and classes.

Use the principles to identify a concrete coupling or substitution problem. A design that follows every letter while adding indirection can be harder to understand than a small cohesive function. The quality question is whether the design keeps requirements, failure behavior, and change cost visible.

## Single Responsibility

The Single Responsibility Principle says that a module should have one reason to change. A reservation service that validates an input, calculates a price, writes SQL, renders HTML, and sends email has several unrelated reasons to change. Separate responsibilities at boundaries that need independent change or testing.

~~~php
<?php

declare(strict_types=1);

final readonly class Reservation
{
    public function __construct(
        public string $id,
        public string $email,
        public int $amountMinor,
    ) {
    }
}

interface ReservationStore
{
    public function save(Reservation $reservation): void;
}

interface Mailer
{
    public function send(string $recipient, string $template, array $data): void;
}

final class ReservationService
{
    public function __construct(
        private ReservationStore $store,
        private Mailer $mailer,
    ) {
    }

    public function create(string $email, int $amountMinor): Reservation
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL) || $amountMinor < 0) {
            throw new InvalidArgumentException('Invalid reservation');
        }

        $reservation = new Reservation(
            bin2hex(random_bytes(8)),
            $email,
            $amountMinor,
        );
        $this->store->save($reservation);
        $this->mailer->send($email, 'reservation-created', [
            'id' => $reservation->id,
        ]);

        return $reservation;
    }
}
~~~

This class still coordinates several collaborators, but persistence and mail delivery are separate responsibilities. Do not split the coordination itself merely to make the class shorter; the use-case transaction is a legitimate responsibility.

## Open for Extension, Closed for Modification

The Open/Closed Principle describes code that can support a new behavior through an extension point without editing stable decision logic. A strategy interface can be useful when new pricing policies arrive independently.

~~~php
<?php

declare(strict_types=1);

interface DiscountPolicy
{
    public function discountFor(int $amountMinor): int;
}

final class NoDiscount implements DiscountPolicy
{
    public function discountFor(int $amountMinor): int
    {
        return 0;
    }
}

final class MemberDiscount implements DiscountPolicy
{
    public function discountFor(int $amountMinor): int
    {
        return min(500, intdiv($amountMinor, 10));
    }
}

final class PriceCalculator
{
    public function __construct(private DiscountPolicy $discount)
    {
    }

    public function total(int $amountMinor): int
    {
        return max(0, $amountMinor - $this->discount->discountFor($amountMinor));
    }
}
~~~

An extension point has a cost: an interface, implementations, configuration, tests, and a way to select the policy. Add it when the variation is real and likely, not for every imagined future rule. A match expression can be simpler when the set is small and controlled.

## Liskov Substitution

The Liskov Substitution Principle requires that a subtype honor the observable expectations of the type it replaces. It must preserve valid preconditions, postconditions, and error meanings that callers rely on. In PHP, satisfying a method signature is not enough.

A repository implementation that silently returns partial records, changes ordering, or throws a different class of failure may violate the caller's contract even though it implements the same interface. Document invariants such as whether a missing row returns null, whether save is atomic, and which exceptions are retryable.

Do not force unrelated concepts into an inheritance hierarchy. A read-only object is not a valid subtype of a mutable collection merely because both contain items. Prefer a smaller capability interface or composition when the behavior contracts differ.

## Interface Segregation

The Interface Segregation Principle says clients should not depend on methods they do not use. A broad UserGateway with password reset, billing, audit, export, and profile methods makes each caller depend on unrelated change. Split interfaces around consumer capabilities when that reduces coupling.

~~~php
<?php

declare(strict_types=1);

interface UserReader
{
    public function find(int $id): ?array;
}

interface PasswordResetter
{
    public function issueToken(int $id): string;
}

final class AccountSummary
{
    public function __construct(private UserReader $users)
    {
    }

    public function show(int $id): array
    {
        $user = $this->users->find($id);
        if ($user === null) {
            throw new RuntimeException('User not found');
        }

        return ['id' => $id, 'name' => $user['name']];
    }
}
~~~

Do not create one interface per class by reflex. An interface has value when it defines a stable boundary, allows a meaningful alternative, or makes a client depend on a smaller capability.

## Dependency Inversion

The Dependency Inversion Principle keeps high-level policy independent from low-level mechanism. A reservation use case should depend on a store capability, while a PDO adapter depends on the database. Dependency injection makes the direction explicit and helps tests replace infrastructure.

Dependency inversion does not require a service locator or a container everywhere. Constructor injection is usually clearer; configure concrete objects at the composition root. Keep the abstraction owned by the consumer when that expresses the consumer's needs rather than exposing every method of the infrastructure.

## Using the Principles Together

The principles interact. A cohesive use case can depend on small capability interfaces, use a strategy for a real variation, and rely on documented behavioral contracts. The result should make a change local and a failure visible.

Use a change scenario to evaluate the design:

1. Identify the requirement that changes.
2. Find the classes and interfaces that must change.
3. Check whether callers depend on unrelated behavior.
4. Verify the replacement preserves error and state contracts.
5. Measure the added indirection and test setup.
6. Keep the smallest design that makes the next change safe.

## Testing and Failure

Test public behavior and boundary contracts rather than private class shape. Contract tests can run every repository adapter against the same expectations. Integration tests prove PDO, transactions, and database constraints; unit tests prove policy branches quickly. A principle that makes testing harder may indicate the abstraction is at the wrong boundary.

SOLID does not remove concurrency, authorization, or transaction concerns. A well-factored service can still have a race, and an interface can still expose a dangerous operation. Keep domain invariants and operational limits explicit.

## Common Mistakes

- Treating SOLID as five mandatory layers for every feature.
- Splitting a cohesive use case until control flow is hidden.
- Adding abstractions for variations that do not exist.
- Calling a class substitutable because its method signatures match.
- Using a service locator and calling it dependency inversion.
- Designing interfaces around infrastructure instead of client capabilities.
- Ignoring transaction, authorization, and error contracts.

## Senior Engineer Thinking

Use SOLID to localize real change and preserve behavior at boundaries. Start from a change scenario, define the smallest useful capability, document substitution contracts, and accept simple concrete code when no variation or independent reason to change exists.

## Exercises

1. Refactor a class that validates input, writes a database row, and sends mail. Identify which responsibilities should remain coordinated.
2. Define a repository contract and run it against an in-memory fake and a database adapter.
3. Find a subtype that narrows a precondition or changes an exception and redesign the boundary.
4. Compare a strategy interface with a match expression for three known pricing rules.

## Review Questions

1. What does “one reason to change” mean in a concrete class?
2. When is an extension point more costly than a conditional?
3. What makes a subtype behaviorally substitutable?
4. Why should interfaces follow client capabilities?
5. How does constructor injection differ from a service locator?
6. Which operational guarantees remain outside SOLID?

## Summary

SOLID provides heuristics for localizing change and preserving behavioral contracts. Use cohesive responsibilities, real extension points, substitutable implementations, focused interfaces, and explicit dependency direction where they solve a concrete problem. Keep transactions, authorization, failure behavior, and operational cost visible, and avoid abstraction without a change-driven reason.

## References

- [PHP interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)
- [PHP dependency injection concepts](https://www.php.net/manual/en/language.oop5.decon.php)
- [Robert C. Martin: The Principles of OOD](https://web.archive.org/web/20160304025545/http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)
- [PHP-FIG PSR-11 Container Interface](https://www.php-fig.org/psr/psr-11/)

