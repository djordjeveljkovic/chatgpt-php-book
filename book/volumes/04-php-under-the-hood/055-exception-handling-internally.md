---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 55
title: Exception Handling Internally
slug: exception-handling-internally
status: complete
summary: ../../_ai/chapter-summaries/055-exception-handling-internally-summary.md
---

# Chapter 55 — Exception Handling Internally

## Why This Matters

An exception is not a jump directly from `throw` to the nearest `catch`. PHP must create or receive a `Throwable`, mark an exception as pending, locate a matching handler, run `finally` blocks while leaving protected regions, unwind call frames, and either resume at a catch or terminate through the global failure path.

This machinery explains why code after `throw` is unreachable, why cleanup belongs in `finally`, why an exception can cross several function boundaries, why a `finally` exception can replace the original failure, and why a stack trace is both useful and potentially expensive or sensitive. It also clarifies the boundary between language semantics and current Zend implementation details.

## Mental Model

```text
throw expression or engine error
              ↓
Throwable object becomes pending
              ↓
search current protected region / call frame
              ↓
run required finally and cleanup paths
              ↓
matching catch? ── yes → bind Throwable and resume catch
       │
       no
       ↓
unwind caller frame and continue searching
              ↓
global handler? ── yes → invoke handler
       │
       no
       ↓
uncaught fatal termination
```

The diagram is a language-level model. The exact handler tables, opcodes, executor globals, and unwinding code are implementation details.

## Core Concept: `Throwable` Is an Object Contract

Userland `Exception` and engine `Error` objects implement `Throwable`. Code can catch the common contract or a narrower class:

```php
<?php

declare(strict_types=1);

try {
    $value = json_decode('{', associative: true, flags: JSON_THROW_ON_ERROR);
} catch (JsonException $exception) {
    echo 'Invalid JSON';
} catch (Throwable $exception) {
    // Boundary-level fallback for unexpected failures.
    throw $exception;
}
```

The thrown value is an object carrying a message, code, file/line information, and a trace according to its class and creation path. The trace is diagnostic context, not a safe place for secrets.

`Error` is not “an error code returned from C”; it is a `Throwable` object used for serious engine/language failures such as type errors. A `catch (Exception)` does not catch every `Throwable`; catch the narrow expected type, or `Throwable` only at a deliberate boundary.

## What PHP Does

When a `throw` executes, statements after it in the current path do not run. PHP searches for the first compatible `catch`, executing intervening `finally` blocks on the way.

```php
<?php

function authorize(bool $allowed): void
{
    if (!$allowed) {
        throw new RuntimeException('not authorized');
    }

    echo 'authorized';
}

try {
    authorize(false);
    echo 'unreachable';
} catch (RuntimeException $exception) {
    echo $exception->getMessage();
}
```

A `finally` block runs after the `try`/`catch` path regardless of whether execution was normal or exceptional:

```php
<?php

$lockHeld = true;

try {
    doWork();
} finally {
    $lockHeld = false;
}
```

Do not use `finally` to hide a failure. A `return` or a new `throw` inside `finally` can replace the result or exception being unwound. If both the protected body and `finally` throw, the `finally` exception is the one propagated, with the earlier exception available as its previous exception where PHP records that relationship.

## What Zend Does: Pending Exception and Handler Resolution

Current php-src keeps executor state for the pending exception and uses executor/call-frame information while handling it. The internal API `zend_throw_exception_internal()` receives a `zend_object *` for the exception and installs it into engine state. The executor then processes exception handling, including a `ZEND_HANDLE_EXCEPTION` path and catch/finally-related opcodes.

Conceptually:

```text
EG(exception) ──> Throwable object
       │
       └── current execute_data
              ├── current function/opline
              ├── previous execute_data
              └── handler/finally metadata for this code
```

This is a source-reading model, not a public API guarantee. The names and flow in php-src are useful when debugging a specific PHP version, but extensions should use supported Zend APIs and should not manipulate executor globals casually.

The compiler records enough information for the executor to identify protected regions and their handlers. At runtime, the engine can leave the current opcode range, run cleanup/finally paths, and move to an outer frame. If no matching handler exists, the exception reaches the global path.

## Exceptions Raised by Engine Operations

Not every failure begins with an explicit userland `throw`. A type mismatch, invalid operation, or an extension using the exception API can create a `Throwable` and set the pending exception state. For example:

```php
<?php

declare(strict_types=1);

function add(int $left, int $right): int
{
    return $left + $right;
}

try {
    /** @var mixed $input */
    $input = 'not an integer';
    add($input, 1);
} catch (TypeError $exception) {
    echo 'Boundary validation failed';
}
```

The engine's failure still participates in the `Throwable`/handler model. Error reporting settings, custom error handlers, and exception conversion can change which failures become exceptions, so test the target PHP version and configuration.

## Matching and Resumption

Catch clauses are considered in source order. A broad catch before a narrow catch makes the narrow branch unreachable for the matching subtype:

```php
try {
    riskyOperation();
} catch (Throwable $exception) {
    report($exception);
} catch (RuntimeException $exception) {
    // Never selected.
}
```

Use the narrow handler first, and use multi-catch when different types have genuinely identical recovery:

```php
try {
    decodeAndStore($payload);
} catch (JsonException|UnexpectedValueException $exception) {
    markPayloadRejected($exception);
}
```

After a matching catch completes normally, execution continues after the catch structure. The engine does not resume at the statement after `throw`.

## Cleanup, Destructors, and Partial Work

Exception unwinding releases local values and leaves frames, but cleanup has multiple layers. `finally` is explicit control-flow cleanup. Destructors are object-lifetime behavior. Database rollback, lock release, temporary-file deletion, and cancellation of external work should be expressed at the boundary that owns them, not left to incidental object destruction.

```php
<?php

$transaction = $database->beginTransaction();

try {
    writeReservation($transaction);
    $transaction->commit();
} catch (Throwable $exception) {
    $transaction->rollBack();
    throw $exception;
}
```

For an object that owns a closeable resource, use `finally` around the operation or an explicit scope abstraction. A destructor may run later, may be delayed by cycles, or may not be a safe place to perform fallible network work.

## Bad Example: Swallowing the Pending Failure

```php
try {
    chargeCard($payment);
} catch (Throwable $exception) {
    return ['ok' => false];
}
```

This converts every failure into the same result, including programming errors, outages, and authorization bugs. It may also leave the caller unable to distinguish “declined” from “payment provider unavailable.”

## Better Example: Translate at a Boundary

```php
<?php

try {
    $receipt = $gateway->charge($payment);
} catch (CardDeclined $exception) {
    return PaymentResult::declined($exception->reason());
} catch (GatewayTimeout $exception) {
    $logger->warning('Payment gateway timeout', [
        'payment_id' => $payment->id,
    ]);
    throw new PaymentTemporarilyUnavailable(previous: $exception);
}
```

The domain boundary translates known failures and preserves the previous exception for diagnosis. Unexpected failures continue to the owning error boundary rather than being silently labelled as a business result.

## Global Exception Handlers

`set_exception_handler()` installs a final userland handler for an uncaught exception at the global scope. It is useful for logging and producing a consistent CLI/HTTP response, but it is not a substitute for local recovery and it does not make an unsafe operation safe.

```php
<?php

set_exception_handler(static function (Throwable $exception): void {
    error_log($exception::class . ': ' . $exception->getMessage());
    http_response_code(500);
    echo 'Internal Server Error';
});
```

Redact messages and traces before sending them to clients. The handler itself can fail; keep it small, defensive, and independent of the subsystem that may already be broken.

## Performance

The normal path through a `try` block is designed to be usable without throwing on every iteration, but actually throwing an exception is substantially more work than returning a scalar result: an object and trace may be created, frames must be inspected/unwound, finally paths may run, and cleanup follows. Exact cost depends on version, build, trace depth, and workload.

Use exceptions for exceptional control flow and domain failures that callers need to handle, not as a substitute for a boolean branch in a tight expected-success loop. Do not remove meaningful exception boundaries based on a synthetic microbenchmark; measure the complete operation, including logging, retries, and recovery.

## Security and Operations

Exception messages, file names, traces, arguments, and previous-exception chains can contain credentials, SQL fragments, tokens, personal data, and internal paths. Log structured metadata with redaction, set an error ID for the client, and keep detailed traces behind access controls.

Retries must be based on failure classification and idempotency. An exception after a network write does not prove that the remote operation did not happen. A catch block that blindly retries can duplicate a payment, reservation, or message. Pair exception handling with timeouts, idempotency keys, transaction boundaries, and durable state.

## Testing

Test the entire unwinding contract:

```php
<?php

final class Probe
{
    public array $events = [];

    public function run(): void
    {
        try {
            $this->events[] = 'try';
            throw new RuntimeException('boom');
        } catch (RuntimeException) {
            $this->events[] = 'catch';
        } finally {
            $this->events[] = 'finally';
        }
    }
}

$probe = new Probe();
$probe->run();
assert($probe->events === ['try', 'catch', 'finally']);
```

Add tests that prove code after `throw` is not run, `finally` executes on both success and failure, a narrow catch is selected before a broad one, previous exceptions are preserved during translation, rollback occurs before rethrow, and a global handler emits a safe response. Test exception paths with injected failures, not only by forcing real outages.

## Common Mistakes

- Catching `Exception` when engine `Error` values must also be handled at the boundary.
- Catching `Throwable` everywhere and hiding programming defects.
- Putting `return` or a new throw in `finally` without understanding replacement behavior.
- Assuming a destructor is equivalent to `finally`.
- Retrying after an exception without knowing whether the remote side committed.
- Logging complete traces and messages into a client-visible response.
- Treating current executor internals as a stable extension ABI.

## Senior Engineer Thinking

For every catch, specify:

```text
What failure classes are expected?
What state may already have changed?
Can the operation be retried safely?
What cleanup does this scope own?
Where is the failure translated, logged, measured, and surfaced?
```

For every throw, identify the recovery boundary. The engine can unwind frames, but only the application knows whether a payment needs reconciliation, a transaction needs rollback, or a client should receive a retryable response.

## Exercises

1. Write nested `try`/`catch`/`finally` blocks and record the exact event order for normal return, caught throw, and uncaught throw.
2. Build a domain exception wrapper that preserves the previous exception and redacts client output.
3. Test a repository method that rolls back on an injected exception and rethrows it. Include a failure during cleanup and decide how it should be surfaced.
4. Design a retry policy for a timeout after an HTTP POST. State how idempotency and reconciliation affect the decision.

## Review Questions

1. What happens to a pending exception as the engine leaves a call frame?
2. Why does `finally` run during unwinding?
3. What happens if `finally` throws while another exception is pending?
4. Why is `catch (Throwable)` appropriate only at deliberate boundaries?
5. Why does an exception after a network write not prove that the write failed?

## Summary

PHP exceptions are `Throwable` objects that become pending when thrown explicitly or by engine/extension operations. The executor searches protected regions, runs `finally` paths, unwinds `execute_data` frames, and resumes at the first matching catch or the global handler. `finally` can replace a return or exception, destructors are not deterministic cleanup scopes, and uncaught handling is a boundary concern. Use typed translation, explicit rollback/cleanup, idempotent retry policies, safe observability, and tests that exercise the complete unwinding path.

## Official References

- [PHP Manual: Exceptions](https://www.php.net/manual/en/language.exceptions.php)
- [PHP Manual: Throwable](https://www.php.net/manual/en/class.throwable.php)
- [PHP Manual: `set_exception_handler`](https://www.php.net/manual/en/function.set-exception-handler.php)
- [php-src: exception throwing and internal handling](https://github.com/php/php-src/blob/master/Zend/zend_exceptions.c)
- [php-src: exception-related execution helpers](https://github.com/php/php-src/blob/master/Zend/zend_execute.c)
- [php-src: exception-related opcodes](https://github.com/php/php-src/blob/master/Zend/zend_vm_opcodes.h)
