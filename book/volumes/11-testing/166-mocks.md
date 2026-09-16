---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 166
title: Mocks
slug: mocks
status: complete
summary: ../../_ai/chapter-summaries/166-mocks-summary.md
---

# Chapter 166 — Mocks

A mock is a test double that verifies an interaction the test has declared in advance. The test says which collaborator should be called, with which arguments, how often, and what it should return. A mock is useful when the interaction is part of the behavior being protected.

Mocks are not automatically better than simpler test doubles. They make a test sensitive to the shape and order of calls, so use them when that sensitivity expresses a real contract. If the result is all that matters, a stub or fake usually gives a more durable test.

## Why this matters

Suppose placing an order must charge the payment provider exactly once and must not send a receipt when charging fails. The payment interaction is a meaningful side effect. A mock can make that rule executable and fail if a refactor charges twice or charges after a rejected order.

The production code should depend on a narrow interface:

```php
<?php

declare(strict_types=1);

interface PaymentGateway
{
    public function charge(int $customerId, int $amountCents, string $idempotencyKey): string;
}

final class OrderService
{
    public function __construct(private PaymentGateway $payments)
    {
    }

    public function place(int $customerId, int $amountCents, string $orderId): string
    {
        if ($amountCents < 1) {
            throw new InvalidArgumentException('Amount must be positive');
        }

        return $this->payments->charge($customerId, $amountCents, 'order:' . $orderId);
    }
}
```

The interface describes the application boundary. It does not expose an HTTP client, SDK object, or database connection to every caller.

## Declaring an expectation

PHPUnit can create a mock and declare its expected interaction:

```php
public function testItChargesOnceWithTheOrderKey(): void
{
    $gateway = $this->createMock(PaymentGateway::class);
    $gateway->expects($this->once())
        ->method('charge')
        ->with(42, 2500, 'order:ord-7')
        ->willReturn('payment-91');

    $service = new OrderService($gateway);

    self::assertSame('payment-91', $service->place(42, 2500, 'ord-7'));
}
```

`expects($this->once())` verifies cardinality and `with()` verifies the argument contract. The mock returns a value so the service can continue its normal path. If the method is called twice, not called, or called with a different key, the test fails at the interaction boundary.

Use a callback when one part of the argument is variable but its invariant is important:

```php
$gateway->expects($this->once())
    ->method('charge')
    ->with(
        42,
        self::greaterThan(0),
        self::callback(static fn (string $key): bool => str_starts_with($key, 'order:')),
    )
    ->willReturn('payment-91');
```

Keep callbacks small. A callback that reproduces production logic creates a second implementation inside the test.

## Failure paths and ordering

A mock can describe failure without contacting a provider:

```php
public function testProviderFailureIsPropagated(): void
{
    $gateway = $this->createMock(PaymentGateway::class);
    $gateway->expects($this->once())
        ->method('charge')
        ->willThrowException(new RuntimeException('provider unavailable'));

    $this->expectException(RuntimeException::class);
    (new OrderService($gateway))->place(42, 2500, 'ord-7');
}
```

If order matters, model that order in a collaborator whose interface makes the sequence meaningful, or use a stateful fake. Tests that assert every incidental call order often fail during harmless refactoring. An ordering assertion is justified when a protocol requires it, such as “begin transaction before write” or “acknowledge only after durable processing.”

## Mocks and asynchronous work

A mock in the request process cannot prove that a queue worker later delivered a message. It can verify that the command was published once, with the right idempotency key and payload. A separate worker test should verify handling, and an integration or contract test should verify the queue adapter.

Avoid asserting a giant serialized payload when only a few fields are part of the contract. Assert stable fields and validate the full schema at the boundary. Include a message ID or deduplication key when retries are expected; “called once” in a unit test does not guarantee once in a distributed system.

## Avoiding overspecified tests

Mock-heavy tests tend to break when they assert private implementation details:

- every helper call rather than the observable outcome;
- exact logging text rather than an event category and fields;
- a particular number of repository queries when the contract allows batching;
- the order of independent calls;
- framework methods that are not part of the application boundary.

Start with the behavior that matters, then mock only the side effect or policy boundary needed to isolate it. A test should explain why an interaction is required. If that explanation is difficult, the production interface may be too broad.

## Testing and operations

Use a real integration test for serialization, HTTP authentication, database transactions, and provider SDK configuration. Mocks verify your caller's decisions; they do not verify that the real adapter accepts the request or that a server honors it.

When a mock expectation fails, inspect whether the production behavior changed or whether the test encoded an accidental implementation detail. Update the interface and its tests together when the domain contract changes. Keep mock failures readable by using named variables and focused expectations.

## Exercises

1. Add a mock expectation that a failed payment never calls a receipt sender.
2. Rewrite an expectation on a full JSON payload so it asserts only the stable contract fields and an idempotency key.
3. Identify an interaction test that is overspecified and replace it with an outcome assertion or a smaller boundary mock.

## Review questions

- What does a mock verify that a stub does not?
- When is an exact call count part of the business contract?
- Why can a mock not prove that an asynchronous job was delivered?
- What makes an expectation overspecified?
- Which boundaries should be covered by an integration test as well as a mock test?

## References

- [PHPUnit documentation: Test Doubles](https://docs.phpunit.de/en/11.5/test-doubles.html)
- [PHPUnit documentation: Mock objects](https://docs.phpunit.de/en/11.5/test-doubles.html#mock-objects)
- [Martin Fowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
