---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 292
title: Performance Investigation
slug: performance-investigation
status: complete
summary: ../../_ai/chapter-summaries/292-performance-investigation-summary.md
---

# Chapter 292 — Performance Investigation

## Why This Matters

Performance problems are rarely solved by making one line of PHP faster. They are solved by measuring the workload, locating the constrained resource, and changing the bottleneck without violating correctness or operational limits. A faster local function can be irrelevant when requests wait for a database connection, lock, queue, or provider.

Chapter 291 established production debugging as evidence-driven investigation. This chapter narrows that discipline to latency, throughput, capacity, concurrency, saturation, and user-visible work. “Make it faster” is not an investigation question.

## Define the Performance Question

Write a brief before changing code:

~~~text
capability:
user-visible target:
workload shape:
baseline and p50/p95/p99:
error budget:
resource constraints:
affected tenants or cohorts:
success and abort criteria:
~~~

Define the user-facing target, measured boundary, known-good baseline, workload, sample period, and guardrails. A p99 API target, a queue-age target, and a memory ceiling are different claims. Name which one is failing.

## Workload and Resource Model

Describe request rate, payload size, data cardinality, concurrency, tenant distribution, cache state, burst behavior, and read/write mix. Distinguish average from peak, throughput from concurrency, warm from cold cache, and synthetic traffic from production behavior.

Decompose a request:

~~~text
queue wait
  + PHP-FPM wait
  + PHP execution
  + database connection wait
  + SQL execution and locks
  + cache and network calls
  + serialization
  + response transfer
~~~

The decomposition is a model, not a promise that every system reports each component. Collect evidence that distinguishes CPU, memory, I/O, lock, connection, and downstream waits. A full FPM pool is a symptom; slow SQL or a provider timeout may be occupying every worker.

## Distributions, Not Averages

Use histograms and percentiles. p50 describes the median experience; p95 and p99 expose the tail experienced by a meaningful minority. Maximums are useful for finding extreme events but are unstable with small samples. Percentiles cannot be combined across instances by averaging their percentile values; aggregate raw observations or use a distribution-preserving method.

Always state sample size, time window, cohort, and boundary. A good average from a warm cache can hide a cold-cache stampede. A service-wide p95 can hide a large tenant or one region with a catastrophic p99.

## PHP-FPM, HTTP, and Worker Limits

Inspect `pm.max_children`, listen queues, worker memory and recycling, request and upstream timeouts, OPcache state, extension/SAPI differences, database connection pressure, graceful reloads, and mixed release populations. `memory_limit` limits PHP-managed allocations for a request; it is not total process RSS or container memory.

For CLI and long-running workers, measure per-job duration, memory trend, connection cleanup, transaction lifetime, worker age, release identity, and restart policy. `memory_get_usage()` and `memory_get_peak_usage()` report PHP-managed memory, not total RSS, native extension allocations, or container pressure. Pair them with process and container measurements.

## Instrumentation and Profiling

Use request timers, traces, sampled CPU profiles, allocation observations, query counts, and dependency timing. A timing record should include outcome, route or job type, deployment identity, and bounded dimensions; it should not log secrets or entire payloads.

Profiling answers different questions at different boundaries:

| Tool or evidence | Question |
| --- | --- |
| PHP profiler | where does PHP execution spend CPU or allocation time? |
| trace | which boundary consumes wall-clock time? |
| query count/timing | is repeated or slow database work dominant? |
| FPM status | are workers waiting, busy, or exhausted? |
| process/container metrics | is CPU, RSS, I/O, or a limit saturated? |
| queue metrics | is work arriving faster than it completes? |

Do not turn on an unrestricted production profiler or payload logging during an incident without assessing overhead, privacy, and blast radius. A diagnostic flag needs an owner, scope, expiry, and rollback.

An illustrative application timer can measure one bounded operation:

~~~php
<?php

function measureOperation(callable $operation): array
{
    $startedAt = hrtime(true);
    $result = $operation();

    return [
        'result' => $result,
        'elapsed_nanoseconds' => hrtime(true) - $startedAt,
    ];
}
~~~

This measures elapsed wall-clock time around the PHP call. It does not attribute database waits, prove a representative workload, or justify logging the result’s contents. In production, pair such observations with bounded outcome and route fields, sampling, and error-safe handling.

## SQL, Plans, and Contention

Inspect query count, rows examined, selectivity, indexes, sort and temporary-table behavior, pagination, lock waits, connection exhaustion, transaction duration, and replica lag. `EXPLAIN` is evidence about a plan under assumptions; it is not proof of observed runtime. Parameter values, statistics, concurrency, database version, cache state, and physical data distribution matter.

Compare estimated and actual execution evidence where the database supports it. Test representative cardinality and skew. Fixing an N+1 query may reveal a connection or serialization bottleneck elsewhere; performance work follows the constrained resource through the system.

## Cache Performance and Isolation

Measure hit rate, miss cost, eviction, key cardinality, hot keys, stampedes, warm-up, invalidation, freshness, and rebuild time. A cache that is faster but omits tenant or authorization scope is a security incident, not an optimization.

Compare cold, warm, and expiry behavior. Include negative lookups and versioned keys. A cache miss that sends every request to an expensive query can produce a thundering herd; a cache fill that holds a lock or stores unbounded payloads can create a different bottleneck.

## Queue and Worker Performance

Measure arrival rate, service rate, depth, oldest-item age, retry amplification, lease expiry, poison messages, and worker concurrency. If arrivals exceed sustained service, backlog grows regardless of average latency. Little’s Law, (L = lambda W), is a useful approximation when the system is sufficiently stable; document its workload and stationarity assumptions rather than treating it as a capacity guarantee.

Increasing consumers may improve throughput while exhausting database connections or a downstream provider. Batching can reduce round trips while increasing memory, retry scope, or tail latency. Measure the whole workflow, not only the worker’s PHP loop.

## Benchmark Design and Traps

Microbenchmarks can isolate a function, but they omit I/O, locks, network, data skew, serialization, and deployment behavior. A useful benchmark states runtime and extension versions, data volume and distribution, cache and OPcache state, concurrency and request mix, warm-up and sample size, measurement boundary, omitted costs, environment, and resource limits.

Beware coordinated omission, insufficient samples, compiler or JIT differences, local-only latency, unrealistic fixtures, and optimizing a synthetic path users do not call. A benchmark result is evidence for its stated workload, not a universal ranking of implementations.

## Optimization Trade-Offs

| Optimization | Possible benefit | New risk |
| --- | --- | --- |
| caching | lower repeated work | stale or mis-scoped data |
| batching | fewer round trips | larger retries and memory usage |
| parallel calls | lower wall-clock latency | downstream load amplification |
| more workers | higher concurrency | database exhaustion |
| denormalization | faster reads | rebuild and consistency cost |
| larger pages | fewer requests | memory and tail-latency growth |

Choose an optimization only after identifying the bottleneck and defining guardrails. A database index can improve reads while slowing writes. A larger page can reduce request overhead while increasing memory and serialization. A local optimization that raises dependency concurrency can worsen total user latency.

## Correctness and Security Constraints

Never optimize away authorization or tenant checks, validation limits, transaction invariants, idempotency, stable ordering, freshness guarantees, or auditability. A cache key must preserve access scope. A pagination shortcut must preserve cursor semantics. A batching optimization must not turn one safe retry into a duplicate multi-effect operation.

Performance can also expose sensitive information through response timing, cache behavior, quota errors, or existence checks. Compare security-sensitive paths carefully and document any accepted timing or freshness trade-off.

## Capacity and Scaling Investigation

Model bottlenecks across PHP-FPM, CPU, memory, database connections and locks, Redis, queues, network calls, and provider quotas. Distinguish vertical scaling, horizontal scaling, batching, caching, backpressure, and workload reduction. More replicas do not help when every replica shares the same connection pool, lock, hot key, or provider limit.

For a capacity claim, record arrival rate, service rate, concurrency, resource ceiling, headroom, and failure behavior. Include noisy tenants and burst traffic. The first saturated resource often changes every downstream measurement, so inspect it before optimizing a later stage.

## Safe Performance Experiments

Use an experiment record containing a hypothesis, predicted signal, baseline, change, scope, duration, guardrails, abort condition, rollback or recovery plan, result, and decision. Run canaries or cohort comparisons where possible. Hold the workload and measurement boundary stable. Guardrails should include errors, correctness, authorization failures, dependency saturation, queue age, memory, and user-visible latency—not just the target metric.

Stop when the abort condition is met, even if the local benchmark improved.

## Rollout, Rollback, and Verification

Roll out query changes, indexes, cache versions, worker counts, batching, and feature flags gradually. Check mixed application, schema, and message versions. An index or schema change may not be removable without cost; a cache-key change may invalidate warm state; a worker-concurrency change may overload a dependency before the application metric moves.

Verify latency distributions, errors, saturation, queue age, database load, memory, correctness, security, and recovery behavior. A successful deployment command proves artifact placement, not performance improvement. Keep the old path or a forward-recovery plan until the evidence supports retirement.

## Case Study: Tenant-Scoped Search

Chapter 287’s search service shows p99 latency rising only for large tenants. PHP-FPM workers become occupied, SQL plans degrade with data size, cache misses trigger repeated queries, and queue-backed indexing falls behind.

Investigate by:

1. defining the affected cohort and user-visible target;
2. separating FPM wait, PHP time, database connection wait, SQL time, cache miss cost, and queue delay;
3. comparing query plans and actual rows for representative tenant sizes;
4. checking cache key version, eviction, stampede, and projection lag;
5. measuring database connections, locks, worker memory, and queue service rate;
6. selecting a bounded change, such as an index, query shape, cache policy, or admission limit;
7. canarying it with correctness, isolation, latency, and dependency guardrails;
8. verifying search ordering, tenant scope, projection convergence, and backlog recovery.

The best answer may be query correction rather than more workers, or workload reduction rather than a larger cache. The investigation earns that conclusion by attribution.

## Common Mistakes

* optimizing a line before defining the workload and failing boundary;
* reporting average latency without p95/p99, sample size, or cohort;
* treating PHP memory metrics as RSS or container usage;
* assuming a full FPM pool proves an FPM defect;
* treating `EXPLAIN` estimates as observed runtime;
* adding consumers until a database or provider is exhausted;
* benchmarking only warm cache or only local execution;
* removing authorization, validation, ordering, or idempotency for speed;
* flushing caches or changing several variables at once;
* declaring success when one metric improves but errors, queue age, or correctness worsen;
* forgetting rollback, forward recovery, and mixed-version behavior.

## Senior Engineer Thinking

Performance investigation is evidence-driven attribution. Define the user-visible question, model the workload, measure distributions, decompose latency, locate the constrained resource, and change one bounded variable. Keep correctness, security, capacity, and recovery as hard constraints. The fastest component is not useful if it makes the whole system slower or less safe.

## Exercises

1. Write a performance brief for a tenant-scoped search endpoint with workload, baseline, p95/p99, and abort criteria.
2. Decompose a slow request into queue, FPM, PHP, database, cache, provider, and transfer time.
3. Compare two query plans using representative cardinality and concurrency assumptions.
4. Design cold-cache, warm-cache, and expiry-stampede measurements.
5. Build a worker-capacity model using arrival rate, service rate, concurrency, and database connection limits.
6. Create an experiment record for adding an index or changing a cache key.
7. Identify which optimizations in the trade-off table could violate tenant isolation or idempotency.
8. Define rollout and rollback verification for a query, worker, or projection change.

## Review Questions

* Why is “make it faster” an inadequate performance question?
* How do latency, throughput, concurrency, capacity, and saturation differ?
* Why are p95 and p99 more useful than averages for many user-facing failures?
* What can PHP memory measurements prove, and what can they not prove?
* Why does a full FPM pool require attribution?
* Why is an estimated query plan not proof of observed performance?
* How can more queue consumers make a system slower?
* What benchmark conditions can produce misleading results?
* Which correctness and security guarantees must optimization preserve?
* What evidence verifies a performance rollout?

## Summary

Performance investigation begins with a measured user-visible question and workload. Use distributions, latency decomposition, PHP/runtime and shared-resource evidence, query plans plus actual behavior, cache and queue metrics, and capacity models to locate the bottleneck. Optimize with explicit trade-offs, correctness and security constraints, bounded experiments, canaries, rollback or forward recovery, and verification across user impact and system health.

## References

- [Chapter 223 — Performance Mental Model](../../volumes/15-performance/223-performance-mental-model.md)
- [Chapter 224 — Measuring Performance](../../volumes/15-performance/224-measuring-performance.md)
- [Chapter 225 — Benchmarking](../../volumes/15-performance/225-benchmarking.md)
- [Chapter 226 — Profiling](../../volumes/15-performance/226-profiling.md)
- [Chapter 229 — Database Performance](../../volumes/15-performance/229-database-performance.md)
- [Chapter 233 — Caching](../../volumes/15-performance/233-caching.md)
- [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md)
- [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md)
- [Chapter 235 — Scaling](../../volumes/15-performance/235-scaling.md)
- [Chapter 261 — Metrics](../../volumes/17-production-engineering/261-metrics.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 291 — Debugging Production](291-debugging-production.md)

## Chapter 293 Handoff

Performance improvements must preserve authorization, data boundaries, and operational safety. Chapter 293 will apply the same review discipline to security architecture, threats, controls, evidence, and response.

The next chapter should begin with:

> Security review is not a checklist of dangerous functions. It is an examination of trust boundaries, principals, assets, abuse paths, controls, and evidence under realistic failure and deployment conditions.
