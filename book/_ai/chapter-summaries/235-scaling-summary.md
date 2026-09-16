# AI Summary — Chapter 235 — Scaling

- Status: complete
- Volume: Volume 15 — PERFORMANCE
- Last updated: 2026-09-16

## Written material

Defines scaling as capacity management against workload, latency, consistency, failure, and cost constraints. Covers vertical and horizontal scaling, stateless PHP-FPM applications and shared state, bottleneck identification, capacity estimates and headroom, database replicas and sharding, connection pools, caches, queues, coalescing, overload handling, autoscaling, failure domains, compatible deployments, observability, and scaling tests. Includes a typed PHP worker-capacity calculator, exercises, review questions, summary, and references.

## Concepts already explained

Effective capacity, vertical scaling, horizontal scaling, stateless application, shared state, read-after-write consistency, read replica, sharding, connection-pool budget, headroom, autoscaling control loop, load shedding, graceful degradation, failure domain, mixed-version compatibility, hot key, queue coalescing, and dependency-aware scaling.

## Terminology established

Capacity target, service rate, concurrency limit, burst factor, capacity model, scale-up delay, dependency saturation, admission control, overload contract, and scale-down protection.

## Examples used

A finite-resource request diagram, a workload-specific scaling table, an idealized connection-capacity calculation, a typed `requiredWorkers()` PHP function, database read-replica and sharding trade-offs, cache and queue scaling guidance, and mixed-version deployment and failure tests.

## Cross-references

[Chapter 229 — Database Performance](../../volumes/15-performance/229-database-performance.md), [Chapter 231 — OPcache](../../volumes/15-performance/231-opcache.md), [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md), [Chapter 233 — Caching](../../volumes/15-performance/233-caching.md), and [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md).

## Open threads

Begin Volume XVI with Chapter 236 on network boundaries.

## Exact next section

Chapter 236 — Network Boundaries: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP example. Local Markdown links resolved and `git diff --check` passed. The capacity formula is explicitly illustrative; no live load, database, cache, queue, or autoscaling test was run.

## Writing notes

The chapter preserves the performance volume's measurement-first approach and distinguishes application replicas from shared dependency capacity.
