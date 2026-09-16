---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 235
title: Scaling
slug: scaling
status: complete
summary: ../../_ai/chapter-summaries/235-scaling-summary.md
---

# Chapter 235 — Scaling

## Why This Matters

Scaling is the process of increasing the work a system can complete while keeping its user-visible and operational contracts within bounds. It is not synonymous with adding servers. A second PHP-FPM host does not solve a database lock, an overloaded provider, an unbounded queue, or a hot cache key.

Start with a requirement: 200 checkout requests per second at p95 below 400 milliseconds, 50,000 imports completed within an hour, or an interactive queue whose oldest message remains below two minutes. Include an error, consistency, and cost target. Without these constraints, “scale” has no testable meaning.

## A Scaling Mental Model

Model a request or job as work moving through finite resources:

```text
arrival
  ↓
edge → PHP workers → database → cache → upstream provider
  ↓          ↓             ↓          ↓
queue     CPU/memory    locks/pool  network
```

Each resource has a service rate and a concurrency limit. A system is sustainable only when the offered workload stays below the effective capacity of every required bottleneck, with headroom for bursts and failures.

For a resource, a rough capacity estimate is:

```text
capacity ≈ useful concurrent workers / average service time
```

If 20 database connections each complete a transaction in 100 ms, the idealized upper bound is about 200 transactions per second for that operation. Real capacity is lower when CPU, lock waits, connection setup, variance, failures, or other query classes share the resource. Treat the calculation as a hypothesis to measure, not as a promise.

Scaling also changes failure behavior. More workers can produce more simultaneous database writes. More replicas can produce more replication lag. More retries can turn a short outage into a traffic surge. Every capacity change needs a dependency and failure analysis.

## Vertical and Horizontal Scaling

Vertical scaling gives one instance more CPU, memory, storage throughput, or network capacity. It is often the simplest first step: fewer network boundaries, simpler deployment, and no need to distribute process-local state. Its limits are instance size, cost, maintenance windows, and a larger failure domain.

Horizontal scaling adds instances or workers. It can increase capacity and availability, but it requires shared or externalized state, routing, deployment coordination, observability, and a plan for uneven load. A stateless HTTP application is easier to scale horizontally because any healthy worker can handle a request.

The choice is workload-specific:

| Constraint | Likely first investigation |
| --- | --- |
| One process is CPU-bound | optimize the hot path, then add CPU or workers |
| Workers are memory-bound | reduce peak memory, recycle safely, then add memory or hosts |
| PHP-FPM workers are all occupied | shorten waits, tune concurrency, then add capacity |
| Database CPU or locks are saturated | improve query/write shape and constraints before adding app hosts |
| A provider quota is reached | reduce calls, batch, cache, or negotiate quota |
| Traffic arrives in bursts | queue bounded work or apply admission control |

Adding application replicas is useful only if the limiting resource is application capacity or if the change deliberately improves availability.

## Stateless PHP and Shared State

Traditional PHP-FPM processes handle requests independently. Local variables disappear at request completion, and a request sent to another replica cannot rely on a previous replica's memory. This is a useful default for horizontal scaling.

State that must survive a request or be visible to another worker belongs behind an explicit durability or sharing boundary:

* sessions may use a shared session store or a deliberately sticky routing policy;
* uploads belong in durable object or filesystem storage, not only in a worker's temporary directory;
* jobs belong in a durable queue;
* locks and idempotency records belong in a store whose atomic operations match the invariant;
* configuration should be immutable per release or loaded through a controlled configuration source.

Sticky sessions can hide a state-design problem and create uneven load. They may be appropriate for a legacy protocol, but document what happens when the selected worker dies. Shared state usually improves failover at the cost of a new dependency and latency.

Do not confuse OPcache shared memory with application state. OPcache can share compiled scripts between workers on a host; it does not make a PHP variable, session, or in-memory cache globally visible. See [Chapter 231 — OPcache](./231-opcache.md) and [Chapter 232 — PHP-FPM](./232-php-fpm.md).

## Find the Limiting Resource

Before scaling, compare the workload with measurements from each boundary:

* request arrival rate, latency percentiles, and error rate;
* active and queued PHP-FPM workers and peak worker memory;
* CPU, run queue, file descriptors, network, and container limits;
* database connections, query latency, rows examined, lock waits, and replication lag;
* cache hit rate, evictions, hot keys, and origin load;
* queue arrival rate, service rate, oldest age, retries, and in-flight work;
* provider latency, rate limits, timeout count, and remaining quota.

A trace or carefully correlated set of metrics should answer where time is spent. If p99 increases while PHP CPU remains low and database lock waits rise, adding PHP replicas may make the contention worse. If all workers are busy on one provider call, reducing that call's timeout or making it asynchronous may help more than adding workers.

Use a load test that represents request mix, payload size, data cardinality, cache state, dependency latency, and concurrency. A single endpoint at a warm cache can prove that endpoint works; it cannot establish whole-system capacity.

## Capacity Models and Headroom

Capacity planning turns measurements into an operating decision. Suppose a service receives a peak of 80 requests per second, 25 percent burst headroom is required, and a worker can complete 12 equivalent requests per second at the target latency. The rough worker requirement is:

```text
ceil(80 × 1.25 / 12) = 9 workers
```

That number is not a PHP-FPM setting. Check that nine workers fit in memory, their database connections fit in the database pool, and their downstream calls fit provider limits. If each request can hold one database connection, the database must support the resulting concurrency plus other workloads and administrative margin.

A small model makes assumptions explicit:

~~~php
<?php

declare(strict_types=1);

function requiredWorkers(
    float $peakRequestsPerSecond,
    float $burstFactor,
    float $requestsPerWorkerPerSecond,
): int {
    if ($peakRequestsPerSecond < 0.0
        || $burstFactor < 1.0
        || $requestsPerWorkerPerSecond <= 0.0
    ) {
        throw new InvalidArgumentException('Invalid capacity inputs');
    }

    return (int) ceil(
        ($peakRequestsPerSecond * $burstFactor)
        / $requestsPerWorkerPerSecond,
    );
}

$workers = requiredWorkers(80.0, 1.25, 12.0);
~~~

The function models only arrival and measured service rate. A production capacity model should also record the target percentile, error budget, worker memory, dependency concurrency, scale-up delay, and the failure mode when capacity is exceeded. Keep the model in documentation or a checked operational artifact; do not pretend that a formula has replaced a load test.

Headroom is capacity intentionally left unused. It absorbs ordinary variation, deployments, failover, and short bursts. Too little headroom produces queueing and emergency scaling; too much may waste money. Choose it from the recovery time and burst shape the system must tolerate.

## Scaling the Database Boundary

The database is often the first shared bottleneck. Improve the work before multiplying clients:

* remove N+1 queries and fetch only the required columns;
* add or adjust indexes based on query plans and write cost;
* make query grain and pagination explicit;
* shorten transactions and lock only what the invariant requires;
* batch compatible operations while bounding transaction size;
* cache genuinely reusable, correctly scoped results;
* separate analytical or bulk workloads when they interfere with interactive traffic.

Read replicas can increase read capacity, but they introduce a consistency choice. A write followed immediately by a read from a lagging replica may not observe the write. Route reads that require read-after-write behavior to the primary or use a session/position-aware policy. Never use a replica to evade a constraint that only the primary can enforce.

Sharding partitions data across database authorities. It can increase aggregate capacity, but cross-shard transactions, unique constraints, joins, rebalancing, backups, and operator workflows become harder. Choose a shard key from access patterns and invariants, not merely from a uniformly distributed identifier. A key that distributes rows evenly may scatter every request across all shards.

Connection pools are concurrency budgets. A pool per application replica can overrun the database even when each pool appears reasonable. Calculate the worst-case total and reserve capacity for migrations, administration, and other services. See [Chapter 229 — Database Performance](./229-database-performance.md).

## Scaling Caches and Queues

A cache scales repeated reads only when keys, freshness, and invalidation remain correct. More cache nodes do not fix an application that creates unbounded keys or a hot key that overloads one shard. Track origin load and cache failure behavior. A cache outage should produce bounded degradation, not an unbounded database stampede. See [Chapter 233 — Caching](./233-caching.md).

A queue scales burst absorption and asynchronous work, not the underlying business operation. If workers consume faster, their database writes or provider calls may become the new bottleneck. Scale consumers using oldest-message age, service rate, downstream saturation, and scale-up time. Separate latency-sensitive and bulk queues so one class cannot consume every worker. See [Chapter 234 — Queue Performance](./234-queue-performance.md).

Coalescing can be more effective than adding capacity. If ten queued updates make only the newest state relevant, replace them with one versioned update when the domain permits. This is a correctness decision: do not coalesce commands whose individual effects matter.

## Overload and Graceful Degradation

When demand exceeds capacity, the system needs a deliberate response:

* reject work at the edge with a useful retry signal;
* shed optional features such as recommendations or expensive enrichment;
* reduce result size or disable costly filters within a documented contract;
* queue work that does not need an immediate response;
* prioritize payments, authentication, or recovery over bulk activity;
* apply per-tenant and per-operation quotas;
* serve boundedly stale data when the product permits it.

Load shedding is safer when it happens before expensive work or resource acquisition. Returning an error after holding a database connection for 30 seconds is not effective admission control. A fallback must be tested for authorization, freshness, and observability; “degraded” must not become a silent data-loss path.

Autoscaling is a control loop. It observes a signal, waits for a decision and startup delay, adds or removes capacity, and observes the result. Queue age or request latency can be useful signals, but noisy thresholds cause oscillation. Use cooldowns, minimum and maximum capacity, scale-down protection for in-flight work, and a dependency-aware upper bound.

## Deployment and Failure Domains

Scaling changes the size and shape of failure domains. Multiple instances on one host do not provide host failure tolerance. Multiple hosts in one zone do not provide zone failure tolerance. Replicas may share a deployment, network, database, or credential authority.

A deployment should preserve compatibility while old and new instances overlap. Add fields before requiring them, accept old messages during the migration window, and remove obsolete behavior only after producers and consumers have moved. A rolling deploy that changes a queue payload or database contract incompatibly can fail even when every new instance passes its isolated tests.

Scale-out also increases observability volume. Use bounded metric labels, sampling for high-volume traces, correlation IDs, and logs that identify release and instance without exposing secrets. A system that cannot distinguish one replica's failures from another's is harder to operate than a smaller, well-instrumented system.

## Testing Scaling Behavior

Test the system at the boundaries where its promises change:

* increase concurrency until latency and errors reveal the first bottleneck;
* fill the database pool and observe request behavior;
* introduce replica lag and verify read-after-write flows;
* make the cache unavailable and check stampede protection;
* grow queue age and verify prioritization and overload behavior;
* terminate a worker during an external side effect and replay the work;
* deploy mixed application versions and consume old and new messages;
* remove an instance, host, or zone according to the availability target.

Use synthetic data with production-shaped cardinality and distributions. Verify business outcomes, not only HTTP status codes or worker counts. A load test that creates ten million fake requests but no realistic locks, payloads, or dependency waits can give false confidence.

## Common Mistakes

* Adding PHP-FPM workers without checking memory or database connections.
* Treating horizontal replicas as a substitute for a stateless design.
* Scaling consumers until a downstream provider or database fails.
* Using read replicas without defining allowed staleness.
* Autoscaling on queue depth while ignoring oldest-message age and service rate.
* Retrying overloaded work, thereby increasing the overload.
* Keeping every cache key forever or allowing an unbounded fallback to the origin.
* Deploying incompatible code and schema changes in one rolling step.
* Measuring only average latency and declaring capacity from a warm, single-endpoint benchmark.
* Assuming multiple processes are independent when they share a database, cache, filesystem, quota, or lock.

## Senior Engineer Thinking

An experienced engineer asks “what becomes the bottleneck next?” for every scaling proposal. The answer should identify the resource, its safe concurrency, its failure behavior, and the measurement that will reveal saturation.

Prefer the smallest change that satisfies the requirement with a clear rollback. A query-plan improvement, smaller payload, bounded queue, or reduced upstream call can create more useful capacity than an additional application tier. When a larger architecture is justified, make the new boundary explicit: who owns data, what consistency is promised, how messages are retried, how the system degrades, and how operators know it is healthy.

Scaling is successful when the whole system meets its contract at an acceptable cost and can recover from the failures that scaling introduces. A larger diagram is not evidence of a more scalable system.

## Exercises

1. Measure a representative PHP-FPM request at increasing concurrency. Record p50, p95, p99, CPU, peak worker memory, database pool wait, and error rate. Identify the first saturation point.
2. Build a capacity table for an endpoint that uses one database connection and one provider call. Include application replicas, total connections, provider quota, headroom, and scale-up delay.
3. Design a read-after-write flow using a primary and read replicas. State which reads may be stale and how the client behaves when the replica is behind.
4. Design overload behavior for a service with interactive requests and bulk exports. Specify what is rejected, queued, degraded, or prioritized.
5. Run a mixed-version deployment test where a new producer sends a message an old consumer can still process. Record the compatibility window and removal condition.

## Review Questions

* What requirement makes a scaling claim testable?
* Why can adding application workers reduce overall capacity?
* Which state must move out of a PHP worker for simple horizontal scaling?
* What does a read replica trade for additional read capacity?
* Why are queue age and dependency saturation better signals than queue depth alone?
* What is effective headroom, and which failures should it absorb?
* How should a system behave when demand exceeds capacity?
* Which database and message compatibility rules are needed during a rolling deployment?

## Summary

Scaling is capacity management under workload, latency, consistency, failure, and cost constraints. Find the limiting resource before adding instances. Keep HTTP state explicit, treat PHP-FPM workers and connection pools as finite budgets, scale databases, caches, and queues according to their actual contracts, and design overload, deployment, and failure behavior deliberately. Validate the result with production-shaped measurements and business outcomes.

## References

- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Google SRE: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [AWS Builders' Library: Avoiding Insurmountable Queue Backlogs](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)
- [PHP manual: PHP-FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PostgreSQL documentation: High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)
