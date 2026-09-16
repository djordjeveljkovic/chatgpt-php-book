---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 306
title: Performance Checklist
slug: performance-checklist
status: complete
summary: ../../_ai/chapter-summaries/306-performance-checklist-summary.md
---

# Chapter 306 — Performance Checklist

## Why This Matters

Performance work is a measurement and capacity problem. A faster PHP function does not help when the request waits on locks, a saturated connection pool, a cache miss, a queue, or a downstream provider. Use this checklist to connect user-visible targets to the resource that limits them.

## Define the Target

- Name the capability, workload, cohort, and baseline.
- Set latency percentiles, throughput, error, freshness, and cost targets.
- Include normal, peak, large-tenant, cold-cache, and degraded-dependency cases.
- State correctness and security constraints that optimization may not weaken.
- Set a guardrail and abort condition before changing production behavior.

## Decompose the Path

Measure request queueing, PHP execution, allocations, SQL and lock wait, cache, network, provider, queue, serialization, and response transfer. Compare FPM, CLI, and worker behavior. Segment by route, tenant size, query shape, version, and feature flag; averages can hide a failing cohort.

## Resource Checklist

| Resource | Evidence | Typical action |
| --- | --- | --- |
| PHP CPU | profiler, CPU saturation, wall/CPU split | simplify hot work or scale carefully |
| PHP memory | peak/RSS, worker restarts, allocation profile | stream, bound, release, or resize |
| FPM capacity | busy workers, queueing, process memory | tune with DB/provider headroom |
| database | plans, rows, locks, connections, wait | query/index/schema/concurrency change |
| cache | hit rate, key cardinality, stampede, freshness | scope, warm, coalesce, or remove |
| queue | arrival/service rate, age, retries | backpressure, capacity, poison handling |
| provider/network | phase timings, quotas, timeouts | deadline, batching, async, or fallback |

## Experiments

Write a hypothesis, prediction, variable, population, duration, sample size, and success criterion. Compare a representative baseline. Change one meaningful variable where possible, keep guardrails for errors and invariants, and record whether the result applies to warm or cold state.

Do not call a local microbenchmark production evidence. Recheck data shape, concurrency, cache state, PHP version, extensions, database plan, provider behavior, and cost. A faster experiment that increases downstream concurrency may worsen the system.

## Capacity and Backpressure

Estimate arrival rate, service rate, concurrency, resource limits, retry amplification, and headroom. More FPM workers or queue consumers can exhaust connections, increase locks, amplify provider calls, and raise memory pressure. Bound queues, batches, fan-out, response size, and worker lifetime. Define what is rejected, delayed, degraded, or shed under overload.

## Correctness and Security Gates

Before accepting a performance change, verify tenant predicates, authorization, stable ordering, freshness, idempotency, transaction semantics, auditability, and data completeness. Never remove isolation or validation for speed. Ensure caches, projections, indexes, and coalescing keys preserve policy scope.

## Rollout and Regression

Use a canary or bounded cohort when risk permits. Compare percentiles, error rates, saturation, queue age, freshness, and business outcomes against a baseline. Define rollback and forward-recovery limits, especially for schema, cache, projection, and message changes. Remove an optimization if evidence shows its complexity costs exceed its benefit.

## Checklist

- [ ] target, workload, cohort, and baseline are written;
- [ ] p50/p95/p99 and error targets are explicit;
- [ ] latency is decomposed across PHP, SQL, cache, queue, and provider;
- [ ] memory, connections, locks, quotas, and queue capacity are measured;
- [ ] correctness, tenant, authorization, and freshness gates are tested;
- [ ] experiment guardrails and abort conditions exist;
- [ ] rollout, rollback/forward recovery, and owner are named;
- [ ] regression telemetry and review date are recorded.

## Common Mistakes

- optimizing before defining the target;
- using averages only;
- benchmarking unrealistic or warm-only data;
- adding workers without checking shared capacity;
- measuring PHP time while ignoring waits;
- trading memory growth for speed in long-lived workers;
- removing a predicate or authorization check;
- declaring victory without a bounded rollout or regression signal.

## Exercises

1. Decompose a p99 regression in the Chapter 287 search service.
2. Build a capacity model for FPM workers and database connections.
3. Design a cold-cache experiment with correctness and cost guardrails.
4. Set queue backpressure and overload behavior for an importer.
5. Write a canary and rollback plan for a new projection.

## Review Questions

- Which resource actually limits the capability?
- Why are percentiles and cohorts necessary?
- How can more concurrency reduce throughput?
- What makes a benchmark representative?
- Which correctness gates must performance work preserve?
- When is forward recovery safer than rollback?

## Summary

Performance checklists connect targets to workload, latency percentiles, PHP/runtime behavior, SQL, caches, queues, providers, shared capacity, correctness, security, rollout, and recovery. Measure the limiting resource, experiment with guardrails, and verify both user outcomes and system headroom.

## Chapter 307 Handoff

Performance changes must preserve trust boundaries and availability. Chapter 307 provides a security checklist organized around identity, authorization, inputs, tenants, secrets, files, outbound requests, dependencies, queues, logs, abuse, and recovery.

## References

- [Chapter 223 — Performance Mental Model](../15-performance/223-performance-mental-model.md)
- [Chapter 226 — Profiling](../15-performance/226-profiling.md)
- [Chapter 235 — Scaling](../15-performance/235-scaling.md)
- [Chapter 256 — PHP-FPM](../17-production-engineering/256-php-fpm.md)
- [Chapter 292 — Performance Investigation](../20-senior-engineering/292-performance-investigation.md)
- [Chapter 287 — Search/Filtering Service](../19-small-engineering-projects/287-search-filtering-service.md)
