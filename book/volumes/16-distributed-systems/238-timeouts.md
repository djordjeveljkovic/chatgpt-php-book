---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 238
title: Timeouts
slug: timeouts
status: complete
summary: ../../_ai/chapter-summaries/238-timeouts-summary.md
---

# Chapter 238 — Timeouts

## Why This Matters

A timeout is a resource and correctness boundary. Without one, a PHP-FPM worker, database connection, queue lease, or client socket can remain occupied until a lower layer or the user gives up. With an arbitrary timeout, a caller can abandon work that the receiver continues performing, creating duplicates or overload.

Choose timeouts from the operation's contract and dependency behavior. A timeout should leave the system able to recover, tell the caller what is known, and fit inside the parent deadline. It should not be selected only because a particular library requires an integer.

## Types of Timeout

Different phases need different limits:

* **connect timeout:** how long to establish a connection;
* **TLS or handshake timeout:** how long to negotiate a secure protocol;
* **write timeout:** how long to send a request body;
* **read or idle timeout:** how long to wait for response progress;
* **total deadline:** the maximum elapsed time for the complete operation;
* **queue visibility or lease timeout:** how long a worker may hold a message before redelivery;
* **lock timeout:** how long to wait for a contested resource.

A read timeout is not necessarily a total timeout. A streaming response that sends one byte periodically can keep resetting an idle timer while consuming a worker indefinitely. Pair phase limits with an absolute deadline and maximum bytes or items.

## Deadlines, Not Just Durations

A duration is relative to the moment it is applied. A deadline is an absolute point on a monotonic timeline:

```text
request starts ─────────────── parent deadline
       ├─ database ─┤
       └──── provider ─────┘
```

Pass the remaining time to nested operations. If a parent has 250 ms remaining and serialization or cleanup needs 20 ms, a child should not consume all 250 ms. A child timeout that is longer than the parent is harmless only if the parent can cancel it and release resources; many blocking clients cannot do that automatically.

The deadline should be propagated across an asynchronous boundary as a protocol-level expiration policy, such as a remaining TTL or wall-clock expiry with a documented clock-skew allowance. A process-local monotonic timestamp cannot be compared safely by another process or host. A queued job must not execute hours later using a request's stale “five-second timeout” as if it were still meaningful.

## A Typed Deadline Helper

Keep deadline arithmetic in one small value object:

~~~php
<?php

declare(strict_types=1);

final readonly class Deadline
{
    private function __construct(private int $atNs)
    {
    }

    public static function afterMilliseconds(int $milliseconds): self
    {
        if ($milliseconds <= 0) {
            throw new InvalidArgumentException('Deadline must be positive');
        }

        return new self(hrtime(true) + $milliseconds * 1_000_000);
    }

    public function remainingMilliseconds(): int
    {
        return max(0, (int) ceil(($this->atNs - hrtime(true)) / 1_000_000));
    }

    public function expired(): bool
    {
        return $this->remainingMilliseconds() === 0;
    }
}

function callWithDeadline(Transport $transport, Deadline $deadline): string
{
    $remaining = $deadline->remainingMilliseconds();
    if ($remaining === 0) {
        throw new TimeoutException('Parent deadline expired');
    }

    return $transport->request(timeoutMilliseconds: $remaining);
}
~~~

`Transport` and `TimeoutException` are application-level illustrative types. The adapter converts milliseconds to the client library's documented options and checks the deadline again before expensive work. Use integer arithmetic carefully; a real implementation should also guard multiplication overflow if it accepts very large values.

## Timeout Does Not Mean Cancellation

Consider this sequence:

```text
client sends charge request
server commits charge
network response is delayed
client timeout fires
client sees “unknown”
```

The correct result is unknown, not automatically failed. Query the operation by its identity, use a provider idempotency key, or reconcile from a durable record. Issuing a second charge because the first response timed out is unsafe.

Cancellation is a separate protocol capability. Closing a socket may stop the client from waiting, but the server may already have read and processed the request. If cancellation matters, define it in the protocol and make server-side cancellation atomic with the operation's state transition where possible.

## Choosing Values

Use measurements from the dependency's latency distribution and the user-visible budget. A timeout that is shorter than normal p99 creates avoidable failures; one much longer than the request contract consumes scarce workers during an outage. Include connection reuse, cold starts, geographic path, payload size, and overload behavior in the measurement.

Set related limits coherently when the outer caller owns the larger budget:

```text
downstream operation timeout < application total deadline
application total deadline < web-server request timeout
web-server request timeout < external caller deadline, where possible
queue visibility timeout > maximum handler attempt + renewal margin
lock timeout < request or job deadline
```

These are relationships to reason about, not universal numeric rules. The outer-boundary ordering changes when a different system owns the client contract; document that ownership explicitly. If the queue lease is shorter than the handler's normal work, duplicate delivery can occur while the first worker is still running. If the web server kills the request before the application can classify a provider timeout, the outcome and telemetry become less useful.

## Timeout Storms

When a dependency slows, every caller can wait until its timeout. The waiting work consumes PHP workers, connections, memory, and queue slots. If all callers retry at the same moment, the dependency receives more load while unhealthy.

Protect the system with bounded concurrency, circuit breaking, admission control, deadlines, and carefully classified retries. Fail optional work early. Do not increase a timeout to make a failing dependency “more reliable” without checking the capacity cost.

## PHP and Resource Cleanup

The timeout exception must not skip cleanup. Roll back open transactions, release locks, close response bodies, return connections to the pool, and reset request-scoped context. In a long-running worker, a `finally` block is part of correctness, not style.

~~~php
function runAttempt(callable $operation, callable $cleanup): mixed
{
    try {
        return $operation();
    } finally {
        $cleanup();
    }
}
~~~

The cleanup itself needs a bounded policy. A stuck cleanup operation cannot be allowed to defeat the parent deadline indefinitely. Some resources are released by process termination; relying on termination should be a last-resort recovery boundary, not the normal path.

## Testing

Test each phase and the total limit with a controllable fake transport: delayed connect, delayed headers, slow body, no progress, malformed response, and completion just before and after the deadline. Verify that timers use a monotonic clock and that cleanup occurs on every timeout path.

Test the ambiguous outcome explicitly. Make the fake server record a side effect, delay the response, let the client time out, and then query the operation. The application should not duplicate the effect.

In integration tests, use short but nonzero limits and avoid relying on a busy shared network to fail at a precise instant. Inject a clock or deadline source when deterministic boundary tests matter.

## Security

Timeouts limit denial-of-service impact but do not replace authentication, authorization, input limits, or rate limits. A request can be small and still expensive. Protect connection pools and expensive endpoints separately, and avoid revealing internal timeout values or topology in public errors.

## Common Mistakes

* Setting only a socket read timeout and assuming the operation is bounded.
* Giving every nested dependency the original full request timeout.
* Treating timeout as proof that no remote side effect occurred.
* Letting a queue lease expire during normal handler execution.
* Retrying immediately after a timeout without checking duplicate safety.
* Forgetting transaction, lock, stream, or tracing cleanup.
* Using wall-clock time for deadline arithmetic.
* Letting cleanup or logging block longer than the operation's contract.

## Senior Engineer Thinking

The key question is “what resource is released when this timer fires, and what fact remains unknown?” A timeout is valuable when it bounds resource occupancy and produces a recoverable state. It is dangerous when it merely hides an unfinished remote operation behind a local exception.

## Exercises

1. Define connect, read, total, database-lock, and queue-lease limits for one request. Explain their relationships.
2. Simulate a provider that commits a result and delays its response. Design the recovery query and idempotency record.
3. Add deadline propagation to a service that calls a database and two providers in parallel. Reserve time for response serialization.
4. Test a long-running worker for cleanup after a timeout and verify that tenant and tracing context do not leak.

## Review Questions

* Why are phase timeouts and an absolute deadline both useful?
* What does a client timeout leave unknown?
* How can a short queue lease create duplicates?
* Why can increasing a timeout reduce capacity during an outage?
* Which resources must be cleaned up on timeout?
* How does a deadline differ from a duration passed to each child call?

## Summary

Timeouts bound resource occupancy and define what a caller can know after waiting ends. Use phase limits plus one propagated monotonic deadline, relate them to worker, queue, lock, and server limits, clean up every resource, and treat a timed-out side effect as potentially completed. Test delayed and ambiguous outcomes, not only a thrown exception.

## References

- [PHP manual: `hrtime`](https://www.php.net/manual/en/function.hrtime.php)
- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [AWS Builders' Library: Timeouts, retries, and backoff](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Chapter 141 — Idempotency](../../volumes/09-http-and-application-development/141-idempotency.md)
