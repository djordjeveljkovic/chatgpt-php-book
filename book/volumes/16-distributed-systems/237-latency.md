---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 237
title: Latency
slug: latency
status: complete
summary: ../../_ai/chapter-summaries/237-latency-summary.md
---

# Chapter 237 — Latency

## Why This Matters

Latency is the time between a user or producer asking for work and receiving the result promised by the contract. It affects whether a page feels responsive, whether a queue meets its service level, and whether a timeout or retry begins. Latency is not just network distance: it includes queueing, PHP execution, database work, serialization, and waits for scarce resources.

An average can hide the requests that matter. A service with a 40 ms average and a 4 second p99 may be unacceptable for checkout. Measure p50 for typical behavior and p95 or p99 for the tail, then connect slow observations to traces and dependency metrics.

## Mental Model

For a mostly serial request:

```text
edge + queue
  + PHP boot + application CPU
  + database wait + upstream wait
  + serialization + response transfer
```

Parallel calls reduce wall-clock time only when their resources are available:

```text
                ┌─ inventory ─┐
request ────────┤              ├─ join → response
                └─ pricing ───┘
```

The join waits for the slowest branch and must handle partial results. Parallelism can also increase connection, CPU, memory, and rate-limit pressure. A lower latency for one request is not an improvement if it makes the tail or error rate worse at the target concurrency.

## Latency Distributions

Percentiles describe a distribution, not a guarantee for every request. If p99 is 800 ms, approximately 99% of observations are at or below that value in the measured population; ties at the boundary make “one request in a hundred” only an approximation. The population and time window matter; aggregate percentiles across unrelated routes can hide a single bad operation.

Record dimensions that explain variation without creating unbounded cardinality:

* route template and operation name;
* response status and outcome category;
* payload-size bucket and result-size bucket;
* dependency and cache outcome;
* deployment version and worker type;
* tenant class or region when that distinction is operationally meaningful.

Do not use raw user IDs, complete URLs, tokens, or exception text as metric labels. Keep a safe correlation ID for trace lookup instead.

## Queueing and Tail Latency

When utilization approaches the service limit, small bursts wait. A request's service time may remain constant while its observed latency rises because it waits for a PHP-FPM worker, database connection, lock, queue consumer, or upstream quota.

Tail latency compounds across dependencies. A request that calls three services succeeds within its target only when all required branches do. If each branch has an independent 99th-percentile success probability of about 99 percent, the chance that all three are inside their p99 is approximately (0.99^3), or about 97 percent. This is an intuition, not a production SLO calculation; correlated failures and shared bottlenecks change the result.

Reduce fan-out, make optional calls asynchronous, cache stable data, or set an explicit degraded response. Do not hide an unbounded dependency chain behind a larger request timeout.

## Measurement in PHP

Use a monotonic clock for elapsed time. Wall-clock time can jump because the system clock is adjusted; `hrtime(true)` is intended for high-resolution elapsed-time measurements.

~~~php
<?php

declare(strict_types=1);

/** @return array{value: mixed, elapsedMs: float} */
function measureElapsed(callable $operation): array
{
    $started = hrtime(true);

    try {
        $value = $operation();
    } finally {
        $elapsedMs = (hrtime(true) - $started) / 1_000_000;
    }

    return ['value' => $value, 'elapsedMs' => $elapsedMs];
}
~~~

The operation's result is returned so measurement does not accidentally remove work. Production instrumentation should also record success or failure and include a stable operation name. Keep metrics recording bounded and ensure an observability failure does not break the business operation unless it is itself a required record.

Measure both wall time and the relevant component time. A PHP CPU profile can show little CPU while the worker spends most of its wall time waiting on SQL. A database span can be fast while the request waits for a connection or lock before the query begins.

## Latency Budgets

An end-to-end target must be divided into budgets:

```text
client-visible target: 800 ms
  edge and routing:     80 ms
  PHP/application:     180 ms
  database:            220 ms
  upstream:            200 ms
  response margin:     120 ms
```

These numbers are planning allocations, not a license for every component to consume its full budget independently. Serial work adds; parallel work shares a parent budget; retries consume the same overall deadline. Leave margin for queueing, scheduling, and cleanup.

Pass a remaining deadline rather than giving every downstream call a fresh timeout. See [Chapter 238 — Timeouts](./238-timeouts.md). A caller with 100 ms remaining cannot safely start a provider operation with a 2 second timeout.

## Reducing Latency

Choose an optimization from evidence:

* reduce algorithmic work or avoid repeated parsing;
* reduce payload size and serialization cost;
* remove N+1 queries and unnecessary round trips;
* add an index or change query shape after inspecting the plan;
* cache stable, correctly scoped results;
* batch compatible calls;
* parallelize independent calls within a bounded resource budget;
* move nonessential work to a queue;
* reduce lock duration and contention.

Do not optimize a 2 ms PHP loop when the request spends 500 ms waiting for a provider. Conversely, do not add a cache to hide a security-sensitive authorization query without defining freshness and invalidation.

## Cold and Warm Paths

Cold latency can include process startup, class loading, OPcache misses, connection setup, DNS, TLS, empty caches, and database page reads. Warm latency omits some of these costs. Report which path was measured.

PHP-FPM and OPcache change startup costs, but they do not remove database or network waits. A long-running worker may have warm connections and retained state, introducing different latency and leak behavior. Compare like with like: cold CLI command, warm FPM request, queue worker after many messages, or a newly started container.

## Bad and Better Parallelism

This code starts unlimited work and waits for every branch:

~~~php
foreach ($providerIds as $providerId) {
    $responses[] = $client->get('/price/' . $providerId);
}
~~~

It may hold a worker and connections for an unbounded number of providers. A better design caps concurrency, sets one parent deadline, and classifies optional failures. The exact mechanism belongs to the client or async runtime; the policy should remain explicit:

```text
max parallel providers = 4
parent deadline         = 600 ms
optional branch failure = omit recommendation
required branch failure = fail the operation
```

If the provider count exceeds the cap, batch or paginate the work. Parallelism is a resource policy, not merely a syntax choice.

## Testing and Operations

Test latency with representative data size, cache state, concurrency, dependency behavior, and failure injection. Record distributions rather than only the fastest response. Test slow database queries, lock waits, provider delays, connection-pool exhaustion, large responses, and partial branch failure.

Use traces to find the critical path. Compare a slow trace with a normal trace and ask which span became longer, which span waited before starting, and whether the request was already queued. Alert on user-visible percentiles and error budgets, not on one noisy maximum.

## Common Mistakes

* Reporting only average or fastest latency.
* Measuring a warm local operation and applying it to a cold production path.
* Giving each dependency a full independent timeout.
* Parallelizing every call without a concurrency limit.
* Calling a slow optional provider on the critical path.
* Aggregating unrelated routes into one latency metric.
* Treating a faster median as success when p99 or errors regress.
* Using wall-clock timestamps to calculate elapsed duration.

## Senior Engineer Thinking

Latency work begins with “which user-visible contract is failing, for which workload, and where does the time accumulate?” The best change often removes a round trip or a wait rather than making a local function clever. A senior engineer also asks what the optimization costs in memory, connections, consistency, failure handling, and operational complexity.

## Exercises

1. Trace an endpoint with two required and one optional dependency. Draw its serial and parallel critical paths and assign a deadline budget.
2. Measure a PHP operation with `hrtime(true)` under cold and warm conditions. Record the environment and explain the difference.
3. Build a percentile report by route and response-size bucket. Find a case where the aggregate hides a slow subgroup.
4. Design a bounded parallel fan-out policy for ten provider calls, including branch failure and parent-deadline behavior.

## Review Questions

* Why can average latency be healthy while users experience slowness?
* What does queueing add to latency even when service time is unchanged?
* Why does fan-out worsen tail behavior?
* Which clock should measure elapsed time in PHP?
* How should a downstream timeout relate to the parent deadline?
* When is asynchronous work a better latency solution than more PHP CPU?

## Summary

Latency is the whole wait from arrival to the promised result. Decompose it across queues, PHP, databases, network calls, serialization, and response transfer; measure distributions and critical paths; allocate one end-to-end budget; and bound parallelism and fan-out. Optimize the measured bottleneck while protecting tail latency, correctness, and failure behavior.

## References

- [PHP manual: `hrtime`](https://www.php.net/manual/en/function.hrtime.php)
- [Google SRE: Tail at Scale](https://research.google/pubs/the-tail-at-scale/)
- [Chapter 224 — Measuring Performance](../../volumes/15-performance/224-measuring-performance.md)
- [Chapter 238 — Timeouts](./238-timeouts.md)
