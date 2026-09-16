---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 234
title: Queue Performance
slug: queue-performance
status: complete
summary: ../../_ai/chapter-summaries/234-queue-performance-summary.md
---

# Chapter 234 — Queue Performance

Queue performance is the relationship between arrival rate, service rate, latency, capacity, and backlog. A faster worker is useful only when it improves the business objective without exhausting the database, upstream provider, or worker host.

Measure queue age and end-to-end completion time, not only messages per second. A queue can have high throughput and unacceptable user latency, or low depth while silently dropping or rejecting work.

## Why this matters

If messages arrive faster than workers complete them, backlog grows. Little's Law provides a useful approximation:

```text
items in system = arrival rate × average time in system
```

For a stable queue, long-term service rate must exceed arrival rate, with capacity for bursts and failures. Increasing concurrency can move the bottleneck to the database or an external API and increase total latency.

Define the contract first: maximum acceptable age, ordering requirements, retry budget, duplicate policy, priority, retention, and outcome visible to the caller.

## Worker throughput

A worker's effective service rate depends on message cost, I/O waits, batch size, acknowledgment behavior, serialization, and concurrency. Profile representative messages. Separate CPU-bound parsing from database and network wait; the best optimization differs.

A bounded worker loop should make state and failure visible:

```php
<?php

declare(strict_types=1);

function runWorker(Queue $queue, Handler $handler, int $maxMessages): void
{
    for ($processed = 0; $processed < $maxMessages; $processed++) {
        $message = $queue->receive(timeoutSeconds: 2);
        if ($message === null) {
            return;
        }

        try {
            $handler->handle($message);
            $queue->ack($message);
        } catch (RetryableFailure $exception) {
            $queue->release($message, delaySeconds: 5);
        } catch (Throwable $exception) {
            $queue->moveToFailure($message, $exception);
        }
    }
}
```

The queue interface is illustrative. The real transport determines visibility timeout, acknowledgment, redelivery, prefetch, and failure behavior. The handler must be idempotent because a process can fail after the side effect and before acknowledgment.

## Concurrency and bottlenecks

Choose worker concurrency from downstream capacity. A database pool with 20 safe concurrent operations cannot support 200 workers performing one transaction each without queueing or failures. Limit concurrency per queue and per dependency, and use bulkheads for expensive message classes.

Batching can reduce network and transaction overhead, but it increases lock duration, failure scope, memory, and retry complexity. A batch handler must define partial success and replay behavior. Prefetch improves throughput for some brokers but can hurt fairness and increase invisible in-flight age.

Partition by key when ordering or locality matters. A global FIFO queue can reduce parallelism; unconstrained parallelism can violate aggregate invariants. Measure partition hot spots and idle partitions.

## Backlog and overload

A queue absorbs bursts; it cannot create infinite capacity. Apply admission control, quotas, rate limits, and backpressure before the queue grows without bound. Separate interactive work from bulk work so a large import does not starve password resets or payments.

Autoscaling from queue depth alone can create a feedback loop. Use oldest-message age, arrival rate, service rate, worker saturation, dependency capacity, and scale-up time. Do not scale beyond a downstream limit.

When backlog is already large, prioritize work, coalesce obsolete messages, drop boundedly stale jobs, or communicate a pending state. Dropping work must be an explicit product decision with audit and retry implications.

## Retries and poison messages

Retry only classified transient failures. Use exponential backoff with jitter and a maximum attempt or elapsed-time budget. A poison message that fails validation should move to a failure transport without consuming all worker capacity.

A retry can duplicate an external effect. Carry an operation ID or provider idempotency key and persist effect state. A dead-letter replay tool should preserve identity, validate the current schema and authorization, and support a rate limit.

## PHP worker lifecycle

Long-running PHP workers retain process state unlike ordinary request lifecycles. Reset request, tenant, locale, transaction, and tracing context after each message. Bound memory with worker recycling and investigate growth; do not merely raise the memory limit.

Graceful shutdown should stop receiving new messages, finish or release the current message according to the transport, flush telemetry, and exit within a deadline. A forced kill creates a redelivery window that the handler must tolerate.

## Testing and operations

Benchmark representative message mixes with realistic payload sizes and downstream latency. Test duplicates, failures after side effects, visibility expiry, batch partial failure, out-of-order messages, poison inputs, graceful shutdown, and dependency saturation. Use the real transport or a compatible emulator for acknowledgment and redelivery semantics.

Monitor arrival rate, service rate, queue depth, oldest age, end-to-end latency, active workers, retry and failure counts, processing duration, memory, dependency saturation, and dropped or coalesced work. Alert on customer-visible age and failure policy violations.

## Exercises

1. Measure a worker's service rate for small and large messages and estimate capacity for a burst.
2. Design separate queues and worker limits for interactive notifications and bulk exports.
3. Add an idempotency record to a handler and test a crash after the external effect but before acknowledgment.
4. Define a graceful shutdown and replay policy for a poison message.

## Review questions

- Why is queue depth alone an incomplete performance metric?
- How does Little's Law connect throughput and latency?
- Why can more workers reduce system throughput?
- Which costs does batching trade for throughput?
- What makes a retry safe, and what happens to poison messages?
- Which PHP state must be reset in a long-running worker?

## Summary

Queue performance depends on arrival rate, service rate, latency, downstream capacity, and failure policy. Measure age and end-to-end outcomes, tune concurrency and batching against real bottlenecks, isolate priority classes, bound retries and backlog, make handlers idempotent, recycle and drain PHP workers safely, and observe dependencies as well as the queue.

## References

- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [AWS Builders' Library: Avoiding Insurmountable Queue Backlogs](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)
- [Symfony Messenger: Worker messages](https://symfony.com/doc/current/messenger.html#consuming-messages-running-the-worker)
- [PHP manual: Garbage Collection](https://www.php.net/manual/en/features.gc.php)
