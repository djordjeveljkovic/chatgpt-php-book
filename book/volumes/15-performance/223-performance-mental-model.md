---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 223
title: Performance Mental Model
slug: performance-mental-model
status: complete
summary: ../../_ai/chapter-summaries/223-performance-mental-model-summary.md
---

# Chapter 223 — Performance Mental Model

## Why This Matters

Performance is the system's response to workload and constraints. A fast function does not help if a request waits on a database lock, a queue is older than its service level, or PHP-FPM has no free worker. Performance work starts by defining the user-visible outcome, measuring it, and finding the limiting resource.

Optimize for a requirement such as p95 API latency, checkout throughput, import completion time, memory per worker, or infrastructure cost. “Faster” without a workload and target leads to changes that may improve a benchmark while harming reliability.

## Latency, Throughput, and Saturation

Latency is the time for one operation. Throughput is completed work per unit time. Capacity is the sustainable throughput at an agreed latency and error level. As utilization approaches saturation, small bursts create queues and latency rises sharply.

A request's latency is the sum of serial work plus waits:

~~~text
edge + PHP boot + middleware + application
    + database + network + serialization + response
~~~

Parallel work can reduce wall-clock time but increases resource demand and failure coordination. A request that calls three providers in parallel may finish sooner while consuming three connections and handling three timeout paths.

Use percentiles. An average can look healthy while a tail of slow requests causes user timeouts. Track p50 for typical behavior, p95 or p99 for tail behavior, and a maximum or timeout policy. Percentiles are summaries; inspect traces and exemplars for causes.

## Find the Bottleneck

Classify the bottleneck before changing code:

* **CPU:** parsing, sorting, encryption, templates, or PHP loops.
* **Memory:** large arrays, object graphs, copies, leaks, or worker retention.
* **I/O:** database, filesystem, cache, network, or queue waits.
* **Concurrency:** locks, connection pools, PHP-FPM workers, or provider quotas.
* **Coordination:** retries, serialization, contention, or downstream queues.

Amdahl's law gives a useful limit: improving one fraction of a serial operation cannot remove time spent elsewhere. If a request spends 80 percent waiting for SQL, a 2x faster PHP loop yields a small total improvement. Measure the fractions before investing.

## PHP Runtime Effects

PHP-FPM commonly handles requests in separate worker processes, while OPcache avoids repeating some compilation work. Long-running workers and CLI imports retain process memory and state between jobs. PHP arrays are flexible hash tables with more memory overhead than packed domain-specific structures; copying uses copy-on-write until a write requires separation.

These are mental models, not reasons to guess. Measure allocations, query count, worker memory, opcode cache status, and request traces in the actual runtime. Replacing a clear array with a lower-level structure may save memory while making correctness and maintenance worse.

## Budgets and Backpressure

Set budgets at the boundary:

* request deadline and maximum body size;
* database query and lock timeout;
* provider connect and total timeout;
* queue job age and execution limit;
* memory limit and worker restart threshold;
* concurrency and rate limits.

Pass a remaining deadline to downstream calls where possible. A retry must fit the original budget; otherwise a request can wait longer because it failed. Backpressure protects the system when demand exceeds capacity: reject early, queue bounded work, shed optional features, or slow producers.

A cache can reduce repeated work, but it creates freshness, invalidation, stampede, and authorization-key requirements. A queue can smooth bursts, but queue age becomes the user-visible latency. Define the contract rather than hiding work.

## A Small Measurement Boundary

Instrument a use case at a stable boundary:

~~~php
<?php

declare(strict_types=1);

function measure(string $name, callable $operation): mixed
{
    $started = hrtime(true);

    try {
        return $operation();
    } finally {
        $milliseconds = (hrtime(true) - $started) / 1_000_000;
        recordHistogram($name, $milliseconds);
    }
}
~~~

Measure useful dimensions such as route, operation, result, dependency, and payload class, but keep cardinality bounded and redact sensitive values. A timer around a function does not explain whether the time was CPU, SQL, a lock, or a remote call; pair it with traces and dependency metrics.

## Correctness and Performance

Never trade an authorization check, transaction invariant, validation limit, or idempotency guarantee for an unmeasured speedup. A faster incorrect response is an outage or security incident. Use indexes and set-based SQL where the database is the right engine, stream large results, bound input, and avoid N+1 calls.

Performance changes alter failure modes. Batching can create larger retries; caching can serve stale authorization; parallel calls can amplify load; increasing worker count can exhaust the database. Review performance and reliability together.

## Testing and Operations

Use representative data volume and workload shape. Test p95 or p99 under realistic concurrency, cold and warm cache, cache miss, database contention, provider latency, and failure. A microbenchmark does not predict production if it excludes serialization, network, locks, or worker queues.

Monitor latency percentiles, throughput, errors, saturation, queue age, database pool use, memory, CPU, and dependency time. Alert on user-impacting budgets. Keep a baseline before changing code so an improvement is measurable and reversible.

## Failure and Threat Analysis

* **Wrong bottleneck:** optimize CPU while a database lock dominates. Trace waits and dependencies.
* **Tail blindness:** average latency hides timeouts. Track percentiles and max age.
* **Overload:** concurrency exceeds a downstream pool. Apply budgets and backpressure.
* **Cache stampede:** simultaneous misses overload the source. Use request coalescing or bounded refresh.
* **Performance regression:** a query or payload grows with data. Test cardinality and plans.
* **Unsafe optimization:** security or consistency is removed. Treat correctness as a hard constraint.
* **Worker retention:** long-running PHP state grows. Measure memory and restart deliberately.

## Exercises

1. Draw a latency budget for an API request with PHP, database, cache, and provider calls.
2. Measure a use case and classify its time as CPU, I/O, lock, or queue wait.
3. Identify one optimization whose failure mode could affect authorization, freshness, or idempotency.
4. Choose percentiles, saturation signals, and alerts for a queue-backed import.

## Review Questions

1. How do latency, throughput, capacity, and saturation differ?
2. Why can a small queue cause a large tail-latency increase?
3. What does Amdahl's law imply about optimizing a minor fraction?
4. Which PHP runtime differences matter for FPM and workers?
5. Why must retries fit a request deadline?
6. How can a performance optimization create a security or reliability problem?

## Summary

Performance is workload behavior constrained by latency, throughput, resources, and failure. Find the bottleneck with measurements, reason about CPU, memory, I/O, and concurrency, set budgets and backpressure, preserve correctness and security, and validate changes under representative load and tail latency.

## References

- [PHP Manual: hrtime](https://www.php.net/manual/en/function.hrtime.php)
- [PHP Manual: Performance](https://www.php.net/manual/en/intro.performance.php)
- [PHP-FPM Configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [Martin Fowler: Performance](https://martinfowler.com/articles/power-of-two.html)

