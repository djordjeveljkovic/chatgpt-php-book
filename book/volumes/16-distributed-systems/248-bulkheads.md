---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 248
title: Bulkheads
slug: bulkheads
status: complete
summary: ../../_ai/chapter-summaries/248-bulkheads-summary.md
---

# Chapter 248 — Bulkheads

## Why This Matters

A bulkhead isolates resource pools so one workload cannot consume everything. A slow recommendation provider should not occupy every PHP worker needed for login. A bulk export should not use all database connections required by checkout.

Bulkheads trade aggregate flexibility for predictable protection. Capacity reserved for one class may sit idle while another is busy, so choose the boundary from business priority, failure impact, and measured workload.

## Mental Model

```text
shared pool: login | checkout | recommendations | bulk export
             ↓ one failure consumes all capacity

bulkheads: [login] [checkout] [optional] [bulk]
             ↓ failure is contained to a pool
```

Possible pools include PHP-FPM pools, worker processes, database connection pools, provider concurrency, queue consumers, CPU quotas, memory budgets, and rate limits. A pool can be logical even when the infrastructure is shared, but a logical limit does not protect against every physical failure.

## What to Isolate

Isolate work with different:

* business priority or user-visible deadline;
* dependency or failure mode;
* memory and CPU profile;
* tenant or trust level;
* retry and queue behavior;
* operational owner or deployment cadence.

Do not create a pool for every route by reflex. Too many small pools cause fragmentation, idle capacity, and complex tuning. Start with a failure or fairness problem that a shared pool cannot safely handle.

## A Semaphore Boundary

A process-local semaphore can enforce a small concurrency budget in a long-running or asynchronous process:

~~~php
<?php

declare(strict_types=1);

final class Semaphore
{
    private int $inUse = 0;

    public function __construct(private readonly int $capacity)
    {
        if ($capacity < 1) {
            throw new InvalidArgumentException('Capacity must be positive');
        }
    }

    public function tryAcquire(): bool
    {
        if ($this->inUse >= $this->capacity) {
            return false;
        }

        $this->inUse++;
        return true;
    }

    public function release(): void
    {
        if ($this->inUse === 0) {
            throw new LogicException('Invalid semaphore state');
        }

        $this->inUse--;
    }
}

function useLimitedDependency(Semaphore $limit, callable $work): mixed
{
    if (!$limit->tryAcquire()) {
        throw new DependencyBusy('Concurrency budget exhausted');
    }

    try {
        return $work();
    } finally {
        $limit->release();
    }
}
~~~

The example is illustrative and not a cross-process lock. The `finally` block protects the local permit. A distributed or multi-process limit needs an atomic shared mechanism and a lease or recovery policy.

In conventional synchronous PHP-FPM, each worker normally handles one request at a time, so this object does not limit concurrency across workers. Use separate FPM pools, a shared dependency limit, or another process-level mechanism when the protection must span workers.

## Pool Sizing

Start with the protected resource, not the number of routes. If a provider safely accepts 20 concurrent calls and checkout needs 8 reserved slots, recommendations may receive at most the remaining provider budget—or use a separate provider account if that is a deliberate contract.

Every pool consumes a part of a larger budget:

```text
FPM workers ≥ checkout + login + optional + bulk
database connections ≥ each pool's maximum concurrent queries
provider quota ≥ all retry and primary calls
host memory ≥ each worker pool's peak memory + system margin
```

If each application replica has the same limit, calculate the total across replicas. A per-process limit of 10 is not a global limit of 10 when 30 processes run.

## Queue Bulkheads

Separate queues and consumers when work has different priority or resource costs. Give bulk workers a maximum concurrency and a database pool that cannot consume interactive capacity. Keep failure and retry policies separate where a poison or slow message class would otherwise starve healthy work.

Separate queues add routing, deployment, and monitoring work. A priority queue may be sufficient when the broker supports fair scheduling and the resource profile is similar. Test starvation and recovery rather than trusting queue names.

## Tenant Isolation

Per-tenant limits protect fairness and reduce noisy-neighbor impact. They can also create high cardinality and leave many tiny pools idle. Group tenants into tiers or use a bounded token policy when individual pools are too costly.

Tenant limits are not authorization. A user who is within quota still needs permission for the operation. Conversely, an administrative override should be auditable and should not allow one tenant to consume every shared physical resource.

## Bulkheads and Circuit Breakers

The patterns address different risks:

* a **bulkhead** limits how much work can be in flight;
* a **circuit breaker** stops calls after a dependency failure pattern.

Use both when appropriate. A healthy but slow provider can exhaust a bulkhead before its failure threshold opens. An open circuit protects the pool while probes test recovery. See [Chapter 247 — Circuit Breakers](./247-circuit-breakers.md).

## PHP-FPM and Process Pools

Separate FPM pools can provide different users, sockets, process limits, and deployment policies, but they also consume more memory and configuration. A logical semaphore inside one application process cannot prevent another endpoint from using the same FPM worker or database pool.

Long-running PHP consumers can use separate worker groups for memory-heavy and latency-sensitive messages. Recycle workers according to measured retention and keep tenant and tracing state isolated. See [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md).

## Testing

Fill one pool and verify that protected pools still serve their contract. Inject a slow provider, large payload, database lock, queue backlog, worker crash, and pool-store outage. Measure rejected work, latency, fairness, idle capacity, and downstream impact.

Test release after success, exception, timeout, and forced shutdown. Test scaling across replicas; a process-local unit test cannot prove a global concurrency policy.

## Security

Use bulkheads to reduce blast radius, not to bypass authorization. Scope tenant quotas from authenticated identity, protect management and override paths, and avoid leaking another tenant's resource usage. A separate pool may require separate credentials and network policy.

## Common Mistakes

* Sharing one pool among work with incompatible priorities.
* Creating many pools without accounting for idle fragmentation.
* Calling a per-process limit a global limit.
* Forgetting retries and redeliveries in provider capacity.
* Leaking permits after exceptions or timeouts.
* Assuming a logical semaphore isolates physical CPU, memory, or connections.
* Using tenant quotas as an authorization decision.
* Adding a pool without a fairness or failure requirement.

## Senior Engineer Thinking

Ask what must remain available during a failure, which resource is actually isolated, and how much capacity the isolation consumes. A bulkhead is successful when an overload in one class leaves a more important class within its contract, with enough shared headroom to avoid needless rejection.

## Exercises

1. Design bulkheads for authentication, checkout, recommendations, and bulk export. Include worker, database, provider, and queue limits.
2. Calculate total concurrency when five replicas each have a local semaphore of 8.
3. Fill the bulk-export pool and prove that login latency remains within target.
4. Decide where separate FPM pools are justified and where a logical semaphore is enough.

## Review Questions

* What risk does a bulkhead contain?
* Why can too many pools reduce useful capacity?
* What is the difference between a local and global concurrency limit?
* How do bulkheads complement circuit breakers?
* Which resource budgets must be included besides worker count?
* Why are tenant limits not authorization?

## Summary

Bulkheads isolate in-flight work and resource budgets so one workload or dependency cannot consume capacity needed by another. Choose a small number of meaningful pools, size them against total downstream and host limits, release permits reliably, combine them with circuit breakers where useful, and test both containment and idle-capacity trade-offs.

## References

- [Microsoft: Bulkhead pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead)
- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Chapter 247 — Circuit Breakers](./247-circuit-breakers.md)
- [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md)
