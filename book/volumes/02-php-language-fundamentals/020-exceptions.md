---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 20
title: Exceptions
slug: exceptions
status: complete
summary: ../../_ai/chapter-summaries/020-exceptions-summary.md
---

# Chapter 20 — Exceptions

## Why This Matters

An exception is a control-flow signal for an operation that cannot produce its promised result. It is useful because a deeply nested function can stop normal work and let a boundary decide what the failure means. It is dangerous when it becomes a generic substitute for validation, ordinary branching, or a policy.

Reliable exception design answers four questions:

- What promise was impossible to fulfill?
- Which layer can recover or translate the failure?
- What state or side effect exists after the exception?
- Should this operation be retried, rejected, compensated, or reported as a bug?

The syntax is small. The engineering consequences are not.

## Mental Model

Think of a thrown exception as an abrupt edge in the control-flow graph:

~~~text
caller
  -> service
      -> repository
          -> dependency
                 X throws
          <- finally / propagation
      <- catch at a meaningful boundary
  <- response, retry, or process failure
~~~

Normal return values describe expected outcomes. Exceptions describe an operation that did not complete normally. The distinction is a design choice: “email address is already registered” might be an expected application result at one boundary, while “database protocol is corrupt” is exceptional. Do not force either choice universally.

## Core Concept

PHP's throwable hierarchy has two main branches:

~~~text
Throwable
├── Exception
│   ├── RuntimeException
│   └── LogicException
└── Error
    ├── TypeError
    └── ...
~~~

User-defined exception classes normally extend Exception or one of its descendants. Error objects represent engine and programming failures, but they implement Throwable and can be caught at a true boundary when the process can safely respond. Catching Throwable inside ordinary business code is usually too broad.

Throw an object:

~~~php
throw new InvalidArgumentException('Quantity must be positive');
~~~

Catch the narrowest class that the current layer can handle:

~~~php
try {
    $receipt = $gateway->charge($payment);
} catch (PaymentDeclined $exception) {
    return PaymentResult::declined();
}
~~~

The exception's message is for diagnostics, not automatically for end users. A stable error code or domain result should cross a public boundary.

## How It Works

### Propagation

If a matching catch is not found in the current function, PHP unwinds the call stack. Code after the throwing operation is skipped. finally blocks execute while unwinding:

~~~php
function readState(FileStore $store): string
{
    $handle = $store->open();

    try {
        return $store->read($handle);
    } finally {
        $store->close($handle);
    }
}
~~~

The return expression is evaluated, but the finally cleanup still runs. If finally itself returns or throws, it can replace the original result or exception. Avoid returning from finally; it makes failure behavior difficult to see and can discard the original cause.

### Preserve the cause

When translating a low-level failure, retain the original Throwable as the previous exception:

~~~php
try {
    $row = $repository->find($id);
} catch (PDOException $exception) {
    throw new UserLookupUnavailable('User lookup failed', 0, $exception);
}
~~~

The outer layer can expose a stable application-level category while operators retain the original stack and driver detail. Do not leak that detail to a client merely because it is stored in the chain.

### Custom exception taxonomy

Exception classes should communicate categories a caller can act on:

~~~php
final class PaymentDeclined extends RuntimeException
{
    public function __construct(public readonly string $reason)
    {
        parent::__construct('Payment was declined');
    }
}

final class PaymentProviderUnavailable extends RuntimeException
{
}
~~~

The first can become a client-visible business outcome. The second may become a retryable 503 response or a queued retry, subject to idempotency. A class for every sentence is not useful; a class for every meaningful policy boundary is.

## What PHP Does

PHP executes the throw expression, creates or receives a Throwable object, and unwinds frames until a compatible catch is found. Type declarations, failed calls, and some engine conditions may throw Error descendants even when application code did not explicitly write throw. The Exception catch type does not catch Error; a catch for Throwable catches both.

finally is the cleanup mechanism, not a transaction. It can close a file or release a lock through a library, but it cannot undo an email, an external API call, or an already committed database transaction. Exception handling must be designed together with side effects.

PHP has no checked exceptions. The language does not force callers to declare or catch a particular exception. Documentation, naming, types, tests, and static analysis must carry the contract. A method that can throw a domain exception should make that part of its public behavior understandable.

## What Zend Does

The Zend VM represents thrown exceptions as an abrupt change from normal opcode execution to handler lookup and stack unwinding. Traces retain call-site information for diagnosis. Creating and unwinding an exception is substantially more work than returning a scalar or a small result, which is one reason exceptions should not be used as the inner-loop implementation of expected values.

The exact internal representation and optimizer behavior are engine details. The application-level guarantees are propagation, matching catch types, finally execution, and the distinction between Exception, Error, and Throwable.

## Minimal Example

This service uses exceptions for a dependency failure and a value result for a normal business outcome:

~~~php
<?php

declare(strict_types=1);

final class ReservationService
{
    public function __construct(
        private ReservationRepository $repository,
    ) {
    }

    public function reserve(int $courtId, DateTimeImmutable $start): ReservationResult
    {
        if ($courtId < 1) {
            throw new InvalidArgumentException('Court ID must be positive');
        }

        if ($this->repository->isOccupied($courtId, $start)) {
            return ReservationResult::unavailable();
        }

        try {
            $reservation = $this->repository->create($courtId, $start);
        } catch (UniqueConstraintViolation $exception) {
            throw new ReservationStoreUnavailable(
                'Reservation could not be stored',
                0,
                $exception,
            );
        }

        return ReservationResult::created($reservation);
    }
}
~~~

The availability result is expected domain behavior. A storage failure is exceptional because this method cannot claim that the reservation was persisted. In a concurrent system, the final uniqueness constraint or transaction must still decide whether two reservations conflict; an earlier availability read is not enough.

## Practical Example: Boundary Translation

An HTTP adapter can translate categories without exposing implementation details:

~~~php
try {
    $result = $service->reserve($courtId, $start);
} catch (InvalidArgumentException $exception) {
    return response(['error' => 'invalid_request'], 422);
} catch (ReservationStoreUnavailable $exception) {
    report($exception);
    return response(['error' => 'temporarily_unavailable'], 503);
}

if ($result->isUnavailable()) {
    return response(['error' => 'court_unavailable'], 409);
}

return response(['id' => $result->reservation()->id], 201);
~~~

The adapter owns HTTP status and the client vocabulary. The service does not need to know whether its caller is HTTP, CLI, or a queue worker. A framework may provide an exception handler, but the same classification and redaction decisions still apply.

## Production Example: Transactions and External Effects

Consider an order flow:

~~~text
begin transaction
  insert order
  charge card
  commit
send confirmation
~~~

If charging succeeds and the process dies before commit, the application may need reconciliation. If the database commits and the email send throws, retrying the whole method may duplicate the charge or order. An exception does not make the sequence atomic.

A safer design might commit the order and an outbox record in one database transaction, then let a worker send the confirmation. The worker must handle duplicate delivery, and the payment operation needs its own idempotency key. The important lesson is not “always use an outbox”; it is that exception boundaries must be aligned with transaction and side-effect boundaries.

## Bad Example

~~~php
<?php

function getPrice(array $product): float
{
    try {
        return (float) $product['price'];
    } catch (Throwable) {
        return 0.0;
    }
}
~~~

This catches programming errors, treats missing or malformed data as a free product, loses the cause, and makes a dangerous value look successful. It also uses an exception for a problem that should normally be prevented by validating the product shape.

## Better Example

~~~php
<?php

declare(strict_types=1);

function priceFromRecord(mixed $record): int
{
    if (!is_array($record) || !isset($record['priceCents'])) {
        throw new InvalidArgumentException('Product price is missing');
    }

    $price = $record['priceCents'];
    if (!is_int($price) || $price < 0) {
        throw new InvalidArgumentException('Product price is invalid');
    }

    return $price;
}
~~~

The function rejects an invalid boundary value and returns a precise money representation. Its caller can decide whether invalid imported data should skip a row, reject a request, or stop a deployment. It does not silently convert a defect into zero.

## Edge Cases

- Catching Exception does not catch Error or TypeError; catch Throwable only at a boundary that can safely handle both.
- A catch block is selected by type compatibility and order. Put specific catches before general catches.
- A finally block runs during unwinding, but a throw or return from finally can replace the original outcome.
- Do not catch an exception merely to rethrow the same object without adding context or changing policy.
- Do not serialize exception messages or traces into an untrusted response.
- A retryable exception needs a bounded retry policy, a timeout, and an idempotency analysis.
- Exceptions thrown after a database commit cannot roll back the commit.
- Destructors and shutdown behavior are poor places for important recovery logic; process termination and output state may be uncertain.
- A broad catch can hide a coding error and leave partially mutated in-memory state. Prefer small units and discard unsafe state.

## Performance

Returning an expected outcome is generally cheaper and clearer than constructing and unwinding an exception for every iteration. Do not optimize away meaningful exceptions solely because they are not free; first measure the workload and preserve the failure contract.

Avoid exceptions as a parser's normal character-by-character result. Validate predictable input with ordinary branches or a parser result, and reserve exceptions for an invalid operation or an unrecoverable boundary failure. On an error path, trace construction, logging, and remote reporting can dominate the cost; keep diagnostics bounded.

## Security

Exception messages, traces, previous exceptions, and arguments can contain credentials, SQL, paths, personal data, and request content. Keep them in protected diagnostics, redact structured fields, and return stable generic messages to clients. Do not let exception type differences reveal sensitive account existence unless that is an intentional policy.

Do not retry authentication, authorization, payment, or destructive operations merely because an exception occurred. Classify the operation's safety and idempotency first. A fail-open catch around permission checks is a security defect.

## Database Interaction

Catch database exceptions at the adapter or transaction boundary where their categories are known. Translate constraint violations to a domain result only when the constraint maps unambiguously to expected behavior. Translate deadlocks and connection failures to a retryable category only when the whole transaction can be retried safely.

Always define transaction ownership. If a repository starts a transaction and an outer service also assumes ownership, an exception can produce confusing partial behavior. Roll back in the owner, preserve the cause, and never report success before commit has completed.

## Concurrency

An exception does not serialize concurrent requests. Two workers can both pass an availability check, and one may then receive a uniqueness exception. Treat the database constraint as the authority, map the losing result intentionally, and ensure retries do not create another side effect.

In a long-running worker, a caught exception may leave dependency clients, transactions, or mutable service state unusable. Clean up explicitly, clear per-job context, and terminate the worker when the runtime or library cannot guarantee a safe reset.

## Testing

Test both the thrown type and the post-failure state:

~~~php
public function testInvalidCourtIdIsRejected(): void
{
    $this->expectException(InvalidArgumentException::class);

    $this->service->reserve(0, new DateTimeImmutable());
}

public function testUnavailableCourtIsAResult(): void
{
    $this->repository->markOccupied(4);

    self::assertTrue(
        $this->service
            ->reserve(4, new DateTimeImmutable())
            ->isUnavailable(),
    );
}
~~~

Add tests for specific-before-general catch order, previous-exception preservation, cleanup in finally, rollback on failure, no client leakage of traces, retry classification, and duplicate requests. Integration tests should exercise real database constraints and transaction behavior; a unit test that only mocks a repository cannot prove concurrency safety.

## Common Mistakes

- Catching Throwable everywhere and returning a default value.
- Using exceptions for ordinary validation results in a hot loop.
- Catching, logging, and continuing after a transaction or dependency entered an unknown state.
- Throwing generic RuntimeException for every category.
- Replacing the original exception instead of chaining it as previous.
- Returning from finally.
- Retrying non-idempotent side effects after an ambiguous failure.
- Assuming a caught exception means all earlier effects were undone.

## Senior Engineer Thinking

Exception design is boundary design. Inside a component, throw when normal completion is impossible and preserve the invariant that successful return means the promised work is complete. At the boundary, translate a category into a result, response, retry, alert, or process exit.

A senior review asks: what state exists after this throw, who owns rollback, is the operation safe to retry, does the exception reveal sensitive information, and can the next request or job use the same process safely? If those answers are unclear, adding another catch block is not progress.

## Exercises

1. Define an exception taxonomy for a payment adapter with declined, invalid request, timeout, and provider-outage categories. State which are retryable.
2. Implement a boundary that catches domain failures, reports unexpected failures, and returns stable client error codes without traces.
3. Write an integration test proving that a uniqueness violation is handled correctly under two competing reservation attempts.
4. Refactor a function that returns magic defaults after catching Throwable. Preserve expected validation outcomes and make programming errors visible.

## Review Questions

1. What is the difference between Exception, Error, and Throwable?
2. Why is a normal domain result often better than an exception for expected business outcomes?
3. What does stack unwinding skip, and what does finally still do?
4. Why should a translated exception chain its previous cause?
5. Why can an exception after commit not undo a database change?
6. When is catching Throwable appropriate?
7. What must be known before retrying an operation that threw?

## Summary

Exceptions provide an abrupt control-flow path for operations that cannot fulfill their contract. Use specific categories, preserve causes, clean up with finally, and translate only at boundaries that know the caller's policy. Do not use broad catches or default values to hide defects. Align exception handling with transactions, external effects, retries, security redaction, concurrency constraints, and worker lifecycle.
