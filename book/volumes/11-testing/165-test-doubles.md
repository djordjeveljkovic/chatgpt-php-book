---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 165
title: Test Doubles
slug: test-doubles
status: complete
summary: ../../_ai/chapter-summaries/165-test-doubles-summary.md
---

# Chapter 165 — Test Doubles

## Why This Matters

A test double is a controlled replacement for a collaborator. It lets a test supply a known answer, record an interaction, or simulate a failure without waiting on a network, clock, queue, or payment provider. Doubles make boundaries observable, but they can also make tests pass against an imaginary implementation.

Use a double when the real collaborator is slow, nondeterministic, expensive, destructive, unavailable, or owned by another test layer. Keep the real collaborator when its integration behavior is the subject of the test. The choice is about the boundary and the question, not a rule that every dependency must be mocked.

## The Vocabulary

The traditional terms describe intent:

- A dummy is an unused value needed only to satisfy a call.
- A stub returns predetermined data so the test can drive a path.
- A fake is a lightweight working implementation, such as an in-memory repository.
- A spy records calls so the test can inspect what happened afterward.
- A mock has configured expectations that fail when a specified interaction does not occur.

Frameworks often use “mock” for all of these. The important distinction is whether the test asserts a result, a state, or an interaction. Prefer state and result assertions when they express the requirement more directly.

## Depend on a Boundary

Inject a small interface that describes the capability the application needs. Avoid passing a large framework client into a domain service; a narrow boundary makes both production and test implementations clearer.

~~~php
<?php

declare(strict_types=1);

interface ReservationStore
{
    public function save(Reservation $reservation): void;
    public function find(string $id): ?Reservation;
}

final readonly class Reservation
{
    public function __construct(
        public string $id,
        public string $status,
    ) {
    }
}

final class ReservationService
{
    public function __construct(private ReservationStore $store)
    {
    }

    public function confirm(string $id): Reservation
    {
        $reservation = $this->store->find($id);
        if ($reservation === null) {
            throw new RuntimeException('Reservation not found');
        }

        $confirmed = new Reservation($reservation->id, 'confirmed');
        $this->store->save($confirmed);

        return $confirmed;
    }
}
~~~

The production adapter can use PDO or an ORM. The service test does not need to know which. If the interface is difficult to fake, it may be exposing too much database or framework detail.

## Stubs Drive Behavior

A stub supplies the branch the test needs. It should be explicit about the case and should not accidentally become a second implementation of production logic.

~~~php
<?php

declare(strict_types=1);

final class StubReservationStore implements ReservationStore
{
    public function __construct(private ?Reservation $reservation)
    {
    }

    public function save(Reservation $reservation): void
    {
        $this->reservation = $reservation;
    }

    public function find(string $id): ?Reservation
    {
        return $this->reservation?->id === $id ? $this->reservation : null;
    }
}

function testMissingReservation(): void
{
    $service = new ReservationService(new StubReservationStore(null));

    try {
        $service->confirm('missing');
        assert(false, 'Expected an exception');
    } catch (RuntimeException $exception) {
        assert($exception->getMessage() === 'Reservation not found');
    }
}
~~~

The test controls a missing row without a database. A separate integration test must prove how the real repository maps a missing database row. Do not add conditionals to a fake simply to mimic every production branch; that makes the fake another system to maintain.

## Fakes and Spies

A fake stores state in memory and can support several scenarios. It is useful for a domain service whose repository semantics are simple, but it should document any differences from the real implementation, such as uniqueness, transactions, isolation, or ordering.

A spy records meaningful interactions:

~~~php
<?php

declare(strict_types=1);

interface Mailer
{
    public function send(string $recipient, string $template): void;
}

final class RecordingMailer implements Mailer
{
    /** @var list<array{recipient: string, template: string}> */
    public array $sent = [];

    public function send(string $recipient, string $template): void
    {
        $this->sent[] = [
            'recipient' => $recipient,
            'template' => $template,
        ];
    }
}

function testConfirmationSendsMail(): void
{
    $mailer = new RecordingMailer();
    // A real service would receive the mailer and call it after confirmation.
    $mailer->send('user@example.test', 'reservation-confirmed');

    assert($mailer->sent === [[
        'recipient' => 'user@example.test',
        'template' => 'reservation-confirmed',
    ]]);
}
~~~

Assert an interaction when the interaction is the requirement, such as “a confirmation is submitted to the mail queue.” Do not assert incidental call order or private helper calls. A recording fake can often express the same requirement with less coupling than a framework mock.

## Mocks and Interaction Coupling

A strict mock expectation can catch a missing call quickly, but it also binds the test to the current implementation. If a refactor changes two internal calls into one equivalent domain operation, a behavior-focused test should continue to pass. Use interaction assertions for boundaries with side effects, retries, authorization checks, or exactly-once submission where the call itself matters.

When a mock represents an external provider, configure realistic failures: timeout before response, explicit rejection, malformed response, and a response received after the provider committed. The application must classify these cases; a mock that only returns success cannot prove recovery behavior.

Keep mock setup near the scenario and fail with a message that identifies the contract. Do not use a mock to verify a value that the test could observe through the resulting state. Excessive mocking is a design signal: deeply nested setup often means the unit owns too many collaborators or the interface is too broad.

## Time, Randomness, and Files

Clocks, random identifiers, filesystem access, and environment reads are boundaries too. Inject a Clock interface, a random token generator, or a temporary filesystem abstraction when deterministic behavior is important. Do not mock the PHP function globally when a small adapter can isolate it.

A fake clock should advance explicitly in the test. A deterministic token source is suitable for asserting persistence and expiry, but never use the test token generator in production. For filesystem tests, prefer a real temporary directory when filesystem semantics matter; a memory fake cannot detect permissions, path traversal, or atomic rename behavior.

## Double Drift and Verification

Every fake is a maintenance obligation. Compare it with the production adapter through shared contract tests: save then find, missing records, duplicate keys, transaction boundaries, and error mapping. Contract tests for a mail or HTTP client can verify that both a fake and an adapter follow the same application interface.

Keep fake behavior smaller than production behavior and document deliberate differences. A fake that silently ignores authorization, uniqueness, or failure can make unit tests green while integration fails. Run a small set of real integration tests to prove the assumptions the fake cannot represent.

## Common Mistakes

- Mocking every dependency by default.
- Returning a success value from a double that cannot reproduce failure or timeout.
- Asserting private call order instead of a public outcome.
- Sharing mutable doubles between tests.
- Letting an in-memory fake hide database constraints or transactions.
- Using deterministic production randomness because a test double leaked into runtime.
- Failing to update a fake when the production interface changes.

## Senior Engineer Thinking

A double is a model of a boundary, not a guarantee that the boundary works. Use stubs to drive a branch, fakes to model simple state, spies to observe meaningful effects, and mocks only when an interaction is itself contractual. Verify the model against the real adapter at an appropriate integration boundary.

## Exercises

1. Implement an in-memory ReservationStore fake and write shared tests that also run against a database adapter.
2. Replace a strict mailer mock with a recording fake and decide which interaction is truly contractual.
3. Add a fake clock to test session expiry and a provider timeout to test retry classification.
4. Identify a unit with too many mocked collaborators and refactor its boundary into smaller capabilities.

## Review Questions

1. How do a stub, fake, spy, and mock differ in purpose?
2. Why can a fake hide a production defect?
3. When is an interaction assertion more appropriate than a result assertion?
4. Which boundaries benefit from injected clocks or random sources?
5. What does a shared adapter contract test provide?

## Summary

Test doubles control or observe collaborators at a boundary. Use stubs to drive behavior, fakes to model simple state, spies to record meaningful effects, and mocks for contractual interactions. Keep interfaces narrow, avoid implementation coupling, model realistic failure, and verify doubles against real adapters with integration or shared contract tests.

## References

- [PHPUnit test doubles](https://docs.phpunit.de/en/11.5/test-doubles.html)
- [Martin Fowler: Test Double](https://martinfowler.com/bliki/TestDouble.html)
- [PHP FIG PSR-11: Container Interface](https://www.php-fig.org/psr/psr-11/)
- [PHP manual: Anonymous classes](https://www.php.net/manual/en/language.oop5.anonymous.php)

