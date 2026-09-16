---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 27
title: Destructors
slug: destructors
status: complete
summary: ../../_ai/chapter-summaries/027-destructors-summary.md
---

# Chapter 27 — Destructors

## Why This Matters

A destructor is a method that PHP may call when an object is being destroyed. That sounds like the object-oriented version of a `finally` block, but it is not. A destructor can be useful for releasing a resource that is owned by one object; it is a poor place to put business decisions, database commits, or work that must happen at a precise time.

The distinction matters because PHP programs have different lifetimes. A short web request often ends soon after the response is built, while a queue worker may process thousands of messages in one process. An object can also have several references, participate in a reference cycle, or be destroyed during shutdown when other services are no longer available.

## Mental Model

Think of an object as having an owner and a lifetime. `unset($object)` removes one variable binding. It does not necessarily destroy the object. Destruction becomes possible when no ordinary references to the object remain, or when PHP's garbage collector later collects a cycle.

There are therefore three separate events:

1. a variable stops referring to an object;
2. the object becomes unreachable;
3. PHP invokes `__destruct()`.

The first event is immediate. The second and third are not always immediate or predictable. At normal request shutdown, PHP also cleans up remaining objects, but the order in which unrelated objects are destroyed should not be treated as an application protocol.

## Core Concept

The destructor hook is named `__destruct`. It receives no arguments and should not return a value. It is called at most once for a particular object instance. A child class that declares a destructor must explicitly call `parent::__destruct()` when the parent's cleanup is also required; parent destructors are not automatically chained in that case.

A destructor should be small, idempotent where practical, and limited to cleanup of resources owned by the object. It should not throw. If cleanup can fail in a way the caller must observe, expose an explicit method that returns or throws that failure, and use `finally` at the boundary that owns the operation.

## How It Works

Conceptually, the Zend Engine stores an object handle and tracks references to the object. When the object is no longer reachable, the engine invokes its destructor before reclaiming the object's storage. Cycles require cycle collection rather than simple reference counting: object A can refer to B while B refers to A even though neither is reachable from application variables.

This is a conceptual model, not a promise about the exact internal data structures or timing. The engine may also run destructors during request shutdown. At that point code that logs, queries a database, or calls an external service may find that dependencies have already been closed or that output is no longer meaningful.

## What PHP Does

The following example demonstrates aliases rather than relying on shutdown behavior:

```php
<?php

declare(strict_types=1);

final class Marker
{
    public function __construct(private string $name)
    {
        echo "created {$this->name}\n";
    }

    public function __destruct(): void
    {
        echo "destroyed {$this->name}\n";
    }
}

$first = new Marker('one');
$second = $first;

unset($first);       // The object is still reachable through $second.
echo "between\n";
unset($second);      // Now PHP can destroy the object.
echo "finished\n";
```

`$first` and `$second` contain references to the same object identity. Copying an object variable does not clone the object. The destructor therefore runs after the second binding is removed, not after the first `unset`.

## Practical Example

An object that owns a file handle can close it as a safety net. The explicit `close` method remains preferable when the caller can use it:

```php
<?php

declare(strict_types=1);

final class AppendLog
{
    /** @var resource|null */
    private $handle;

    public function __construct(string $path)
    {
        $handle = fopen($path, 'ab');
        if ($handle === false) {
            throw new RuntimeException("Cannot open {$path}");
        }

        $this->handle = $handle;
    }

    public function append(string $line): void
    {
        if (!is_resource($this->handle)) {
            throw new LogicException('Log is closed');
        }

        if (fwrite($this->handle, $line . PHP_EOL) === false) {
            throw new RuntimeException('Writing the log failed');
        }
    }

    public function close(): void
    {
        if (is_resource($this->handle)) {
            fclose($this->handle);
            $this->handle = null;
        }
    }

    public function __destruct(): void
    {
        $this->close();
    }
}
```

The explicit method gives the application a place to report an error and a clear point at which the resource is released. The destructor covers abandoned objects and exceptional paths. In a long-running worker, explicit closure also avoids holding file descriptors until an unpredictable later collection.

## Production Example

Use an explicit scope for a transaction-like operation:

```php
$transaction = $connection->beginTransaction();

try {
    $reservationService->reserve($request);
    $connection->commit();
} catch (Throwable $exception) {
    $connection->rollBack();
    throw $exception;
} finally {
    $connection->closeCursor();
}
```

A destructor on a database wrapper must not be responsible for deciding whether to commit or roll back. If the request dies at an unexpected point, a destructor may run too late, have no usable connection, or conceal the original exception. The owner of the unit of work knows the business outcome and should make the decision.

## Bad Example

```php
final class SendsReceipt
{
    public function __construct(private Receipt $receipt) {}

    public function __destruct()
    {
        $this->mailer->send($this->receipt); // timing, network, and failures are hidden
    }
}
```

This makes garbage-collection timing part of a customer-visible workflow. A test may pass in one process and fail in another. A retry can send twice, and shutdown can attempt network I/O after the application has begun closing down.

## Better Example

Make delivery explicit and keep cleanup separate:

```php
final class ReceiptService
{
    public function __construct(private Mailer $mailer) {}

    public function send(Receipt $receipt): void
    {
        $this->mailer->send($receipt);
    }
}
```

If delivery must be retried, persist an outbox record or enqueue a job with an idempotency key. The destructor is not a queue, transaction manager, or reliability mechanism.

## Edge Cases

- An exception from a destructor is dangerous because it can obscure an exception already in flight. Treat destructor code as non-throwing cleanup.
- `exit` does not make destructor timing a good application contract; shutdown cleanup still has limitations.
- Destructors can run while another object they reference is already partly torn down. Avoid calling methods on unrelated services from a destructor.
- Reference cycles may survive until cycle collection. `unset` one root when you need to make a large object graph eligible for collection.
- A child destructor does not automatically run the parent destructor. Call the parent explicitly when the parent owns cleanup.
- Destructors do not run for objects that never finish construction, because no fully constructed instance was returned to the caller.

## Performance

Destructor work is normally constant-time for a small owned handle, but thousands of destructors can create a visible tail during a worker's cleanup or request shutdown. The important cost is often the resource itself: an open file descriptor, socket, temporary lock, or memory graph held longer than necessary. Release those resources at the narrowest useful scope.

## Security

Do not rely on a destructor to delete sensitive temporary files or revoke authorization. Cleanup may be delayed or skipped by a process termination, and a file can remain readable during that window. Use explicit cleanup, restrictive permissions, operating-system facilities such as private temporary files, and a recovery process for abandoned artifacts.

## Testing

Test the explicit lifecycle, not an exact garbage-collection schedule:

```php
final class AppendLogTest extends TestCase
{
    public function testCloseIsIdempotent(): void
    {
        $log = new AppendLog($this->temporaryPath());

        $log->close();
        $log->close();

        self::assertTrue(true); // No warning or exception is the contract here.
    }
}
```

For destructor coverage, create a narrow test double that records cleanup, remove all references, and use such a test only when the cleanup hook itself is the behavior under test. Do not assert the order of unrelated destructors. Integration tests should verify that a worker can process the next job without leaked handles or stale state.

## Common Mistakes

- Treating `unset` as guaranteed destruction.
- Putting network calls, logging policies, or business actions in `__destruct`.
- Forgetting `parent::__destruct()` in an overriding child destructor.
- Assuming request shutdown has the same dependency availability as normal execution.
- Keeping large object graphs alive in a long-running worker.

## Senior Engineer Thinking

Ask who owns the resource, what event defines successful completion, and how failure is reported. If a cleanup action must happen exactly once, be observable, or be retried, make it explicit and give it a durable or transactional boundary. Use a destructor only as a last line of defense for local, non-business cleanup.

## Exercises

1. Predict the output and explain why the destructor runs when it does for two aliases to one object.
2. Add a `release()` method to a resource wrapper. What state should `release()` leave behind, and how will repeated calls behave?
3. Build a small worker loop that creates a temporary file per job. Measure what happens when files are closed explicitly versus waiting for object destruction.
4. Review a class that sends an email from its destructor. Identify the retry, observability, and duplicate-delivery failures, then redesign its public API.

## Review Questions

1. What is the difference between a variable losing a binding and an object being destroyed?
2. Why are cycles different from ordinary unreachable objects?
3. Why is `finally` usually a better place for deterministic cleanup?
4. When can a destructor be a reasonable safety net?
5. Why is a destructor unsuitable for committing a business transaction?

## Summary

`__destruct()` is a cleanup hook, not a deterministic workflow boundary. Object identity, aliases, cycles, and process shutdown affect when it runs. Keep destructor work small and local, expose explicit lifecycle methods when callers need errors or timing, and test observable behavior rather than garbage-collection order.
