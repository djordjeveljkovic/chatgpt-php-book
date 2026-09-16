---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 169
title: Spies
slug: spies
status: complete
summary: ../../_ai/chapter-summaries/169-spies-summary.md
---

# Chapter 169 — Spies

A spy records calls made to it so a test can inspect the interaction after the system under test has run. Unlike a mock, a spy usually does not require expectations before the call. Unlike a fake, it does not necessarily implement the collaborator's real behavior; it captures evidence about how the collaborator was used.

Spies are useful when the test cares about an emitted notification, audit event, metric, or command but wants to assert it after checking the main outcome. They are also useful for diagnosing interactions without making the setup read like a script of every expected call.

## Why this matters

An order service may return successfully and publish an audit event. The test should assert both the result and the event's stable fields. A spy keeps the interaction visible without requiring the service to expose its internal event-building method.

Define a narrow output interface:

```php
<?php

declare(strict_types=1);

interface Notifier
{
    /** @param array<string, scalar> $data */
    public function send(string $event, array $data): void;
}

final class SpyNotifier implements Notifier
{
    /** @var list<array{event: string, data: array<string, scalar>}> */
    public array $calls = [];

    /** @param array<string, scalar> $data */
    public function send(string $event, array $data): void
    {
        $this->calls[] = ['event' => $event, 'data' => $data];
    }
}
```

A service can use the interface without knowing that a test records it:

```php
final class RegistrationService
{
    public function __construct(private Notifier $notifier)
    {
    }

    public function register(int $userId, string $email): void
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }

        // Persist the user in the real application before announcing success.
        $this->notifier->send('user.registered', [
            'user_id' => $userId,
            'email' => $email,
        ]);
    }
}
```

The test inspects the recorded calls after exercising the service:

```php
public function testRegistrationEmitsAnEventWithStableFields(): void
{
    $notifier = new SpyNotifier();
    (new RegistrationService($notifier))->register(7, 'mina@example.test');

    self::assertCount(1, $notifier->calls);
    self::assertSame('user.registered', $notifier->calls[0]['event']);
    self::assertSame(7, $notifier->calls[0]['data']['user_id']);
    self::assertSame('mina@example.test', $notifier->calls[0]['data']['email']);
}
```

The test asserts the observable contract without asserting private helper calls. If the event is intentionally asynchronous, the spy can represent the publisher; a separate consumer test must cover handling and delivery semantics.

## Spy versus mock

A mock declares expectations before execution and often fails at the unexpected call. A spy records calls and lets the test decide what to inspect afterward. The choice is about the test's purpose:

- use a mock for a required interaction whose absence or excess should fail immediately;
- use a spy for post-condition evidence, diagnostics, or a small set of stable fields;
- use a stub when only a return value is needed;
- use a fake when the collaborator's behavior itself is part of the test.

A spy can still assert call count, order, and arguments after the fact. Do not turn every spy into a second mock by asserting every incidental call. Record only what helps explain the behavior and keep the inspection API simple.

## Recording safely

Spy data is test evidence, not a free pass to capture secrets. Redact tokens, passwords, session IDs, and payment data before recording or assert that the collaborator receives a safe representation. A spy that stores every argument can retain sensitive data in test reports and memory.

For a logger spy, inspect event names, severity, and safe context rather than exact rendered lines:

```php
final class SpyAuditLog
{
    /** @var list<array{event: string, context: array<string, scalar>}> */
    public array $records = [];

    /** @param array<string, scalar> $context */
    public function record(string $event, array $context): void
    {
        $this->records[] = ['event' => $event, 'context' => $context];
    }
}
```

The production logger may add timestamps, JSON formatting, trace IDs, and transport metadata. Those belong in logger integration tests, not in every service unit test.

## Spies and failure behavior

A simple spy succeeds whenever called. That is useful for a notification, but it cannot test a provider timeout. Combine a spy with a controlled stub or fake when the production boundary has both output evidence and failure behavior. Alternatively, use a spy implementation that injects a deliberate exception, but keep that capability explicit:

```php
final class FailingNotifier implements Notifier
{
    public function send(string $event, array $data): void
    {
        throw new RuntimeException('notification unavailable');
    }
}
```

The service test should state whether notification failure rolls back registration, queues a retry, or records a degraded outcome. A spy cannot decide that policy for the application.

## Async and external effects

A spy in the publisher process proves that code attempted to publish. It does not prove that a broker accepted, retained, delivered, or redelivered the message. For an external email or webhook, spy on the application adapter in a unit test, then use a sandbox or contract test to verify the real protocol.

When calls can be concurrent, list order may not be a contract. Assert membership and per-event invariants instead of relying on array position. When retries are expected, assert the idempotency key and acceptable attempt count rather than pretending the system will call exactly once.

## Testing and operations

Use spies for audit evidence and metrics decisions where the interaction is important but the concrete transport is not. Keep a small integration test for the serializer and sink. Reset spies between tests and avoid static global call lists; leaked calls create order-dependent assertions.

If an assertion fails, inspect the recorded calls as diagnostic output, then reduce the assertion to the contract that matters. A useful spy makes a failing test explain what happened without requiring a debugger or a copy of the production logger.

## Exercises

1. Add a spy for a password-reset publisher and assert that it receives a user ID and opaque token ID, never the raw token.
2. Change an exact-order assertion for two independent audit events into a set of per-event assertions.
3. Design a publisher spy and a consumer integration test for a retryable webhook.

## Review questions

- How does a spy differ from a mock in setup and assertion timing?
- What information should a spy avoid retaining?
- Why can a publisher spy not prove message delivery?
- When should a spy assert membership instead of list order?
- Which behavior belongs in an integration test for an external sink?

## References

- [PHPUnit documentation: Test Doubles](https://docs.phpunit.de/en/11.5/test-doubles.html)
- [Martin Fowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- [PHP manual: filter_var](https://www.php.net/manual/en/function.filter-var.php)
