---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 246
title: Backpressure
slug: backpressure
status: complete
summary: ../../_ai/chapter-summaries/246-backpressure-summary.md
---

# Chapter 246 — Backpressure

## Why This Matters

Backpressure is the system's response when producers can create work faster than consumers can safely complete it. Without backpressure, buffers grow, memory is exhausted, queues become ancient, PHP-FPM workers wait longer, and downstream services receive more work precisely when they are already unhealthy.

Backpressure is a contract, not simply a queue setting. Decide whether to reject, delay, shed, degrade, or slow the producer. Make the choice visible to the caller and measurable to operators.

## Mental Model

```text
producer rate > consumer rate
          ↓
bounded buffer fills
          ↓
reject | delay | shed | slow producer
```

A buffer can absorb a finite burst. It cannot make an unstable long-term workload sustainable. If arrival rate remains above service rate, backlog grows until a resource or retention limit is reached.

Backpressure can be applied at several boundaries:

* HTTP admission and request-body limits;
* PHP-FPM worker and connection-pool limits;
* queue publish quotas and bounded queues;
* consumer concurrency and downstream semaphores;
* stream reads and writes;
* per-tenant and per-operation rate limits.

The earliest effective boundary usually protects the most resources.

## Buffering and Queueing

An unbounded PHP array is a buffer whose capacity is hidden in process memory. A queue with unlimited retention moves the bound into storage and makes completion latency less obvious. Prefer an explicit maximum count, bytes, age, or cost.

```text
buffer capacity = 1,000 items or 64 MiB or 30 seconds of age
```

When the bound is reached, choose a policy. Rejecting a new low-priority event may be safer than evicting an unprocessed payment command. Dropping a superseded cache-refresh event may be acceptable when the newest state can be recomputed.

## Admission Control

Admission control decides whether work may enter an expensive path. It can use a semaphore, token bucket, queue capacity, tenant quota, or current deadline. Apply it before opening a database transaction or making a provider call.

An HTTP response should communicate overload according to the API contract. A retryable response without a bounded retry policy can create a storm. Include a safe retry hint where appropriate, and make clients distinguish overload from invalid input.

## A Bounded Admission Gate

The gate below models a process-local limit. It is useful for a long-running process or an asynchronous runtime. A conventional synchronous PHP-FPM worker normally handles one request at a time, so this object does not coordinate concurrent FPM workers. A distributed deployment needs a shared or per-instance policy whose total capacity is understood:

~~~php
<?php

declare(strict_types=1);

final class AdmissionGate
{
    private int $inUse = 0;

    public function __construct(private readonly int $limit)
    {
        if ($limit < 1) {
            throw new InvalidArgumentException('Limit must be positive');
        }
    }

    public function tryAcquire(): bool
    {
        if ($this->inUse >= $this->limit) {
            return false;
        }

        $this->inUse++;
        return true;
    }

    public function release(): void
    {
        if ($this->inUse === 0) {
            throw new LogicException('Gate is not acquired');
        }

        $this->inUse--;
    }
}
~~~

The `tryAcquire()` decision is atomic only within one PHP process. Across replicas, use a broker, database, or coordination primitive appropriate to the invariant. Always release in `finally`; a leaked permit is a capacity leak.

## Streams and Backpressure

Streaming APIs need flow control. A producer writing a large response should not create the entire body in memory when the consumer or socket is slow. Read and write in bounded chunks, honor the transport's progress and failure signals, and stop when the client disconnects.

Generators reduce producer-side memory, but they do not automatically limit a fast downstream operation. The consumer must pull at a safe rate or a bounded intermediate buffer must apply pressure. See [Chapter 15 — Functions](../02-php-language-fundamentals/015-functions.md) for generator semantics; the stream-flow principle here applies to any generator or transport implementation.

## Queues and Workers

A queue provides temporal buffering, while consumer concurrency provides service capacity. Limit publishing when the queue's age or storage budget exceeds the contract. Limit workers when the database, provider, or host is saturated. Scaling consumers without checking downstream capacity is the most common way to move overload rather than remove it.

Separate interactive and bulk work. Priority queues need fairness or aging so low-priority work does not starve forever. When backlog is already large, coalesce obsolete commands, reject new bulk work, or communicate a pending state rather than pretending the queue is immediate.

## Backpressure and PHP-FPM

PHP-FPM has a finite worker pool. A request waiting on a full downstream pool still occupies a worker. Edge rate limits, bounded dependency concurrency, and early rejection protect the FPM pool better than allowing every request to wait until its timeout.

`memory_limit` bounds one PHP process; it is not a global backpressure policy. A host can run out of memory before any one process reaches its limit. Include process count, peak memory, OPcache, web server, and other services in the budget. See [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md).

## Shedding and Degradation

Load shedding should preserve the most valuable work. Define priority, freshness, and correctness:

```text
required authorization → fail closed or use approved emergency policy
payment capture         → preserve and reconcile
recommendations         → omit or use bounded stale data
bulk analytics          → delay or reject
```

Do not shed audit records, security events, or financial transitions without an explicit durable alternative. A degraded response must remain authorized and must say enough for the caller to understand what is missing.

## Feedback Loops and Autoscaling

Autoscaling adds capacity after observing a signal and a startup delay. If producers keep increasing while consumers start, the system can oscillate or overshoot a dependency limit. Use maximum capacity, cooldowns, dependency-aware ceilings, and a ramp-up policy.

Monitor the signal that represents customer impact: oldest queue age, request latency, rejected work, and error rate. Queue depth alone cannot distinguish many cheap messages from a few expensive ones.

## Testing

Fill every buffer and observe the documented result. Test full queue, full connection pool, slow consumer, disconnected stream, exhausted tenant quota, priority starvation, cache stampede protection, and graceful recovery after pressure falls.

Use a fake clock and bounded fixtures for thresholds. Verify that permits release after success, failure, timeout, and process shutdown. Load-test the whole chain so a local gate is not hiding a saturated database.

## Security

Backpressure limits resource abuse, but it must not become an authorization bypass. Apply quotas by authenticated tenant or actor where appropriate, avoid revealing another tenant's queue state, and protect administrative override paths. Rate-limit replay and recovery operations too.

## Common Mistakes

* Using an unbounded in-memory buffer for external work.
* Queueing every request without an age or capacity contract.
* Applying admission control after opening expensive resources.
* Scaling consumers until the database or provider fails.
* Returning a retryable overload response with no client budget.
* Shedding audit, security, or financial work without a durable alternative.
* Leaking a semaphore permit on timeout or exception.
* Calling queue depth a health metric without age or completion latency.

## Senior Engineer Thinking

Ask where pressure is first visible, which resource must be protected, and what work should win when capacity is scarce. A good backpressure policy makes overload bounded and legible. It does not make every request succeed; it keeps the system able to recover and preserves the most important invariants.

## Exercises

1. Define buffer limits and full-buffer behavior for HTTP uploads, email jobs, and payment commands.
2. Add a bounded admission gate to a service and test release on every exit path.
3. Design separate overload policies for interactive, optional, and bulk work.
4. Plot queue age while arrival rate exceeds service rate, then apply rejection or coalescing and compare recovery.

## Review Questions

* Why can a queue hide overload rather than solve it?
* Where should admission control happen?
* Why can more consumers reduce system health?
* Which work may be shed, and which work needs durable preservation?
* Why is a process-local semaphore insufficient for a global invariant?
* Which metrics reveal customer-visible backpressure?

## Summary

Backpressure keeps producers from overwhelming finite consumers and dependencies. Bound buffers by count, bytes, age, or cost; reject, delay, shed, degrade, or slow work deliberately; apply admission early; cap consumers by downstream capacity; protect PHP-FPM and host resources; and test full, slow, and recovering conditions with tenant-aware observability.

## References

- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Reactive Manifesto: Responsive and resilient systems](https://www.reactivemanifesto.org/)
- [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md)
- [Chapter 244 — Queues](./244-queues.md)
