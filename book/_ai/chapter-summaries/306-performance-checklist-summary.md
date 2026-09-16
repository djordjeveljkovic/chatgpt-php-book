# AI Summary — Chapter 306 — Performance Checklist

- Status: complete
- Volume: Volume 21 — REFERENCE
- Last updated: 2026-09-17

## Written material

Chapter 306 provides an evidence-driven performance checklist covering targets, cohorts, percentiles, PHP/FPM, SQL, locks, caches, queues, providers, memory, capacity, backpressure, correctness/security gates, experiments, rollout, regression, exercises, and the Chapter 307 handoff.

## Concepts already explained

Workload; baseline; p95/p99; latency decomposition; saturation; headroom; backpressure; retry amplification; experiment guardrail; correctness gate; canary; forward recovery.

## Terminology established

Tenant search p99, FPM/database capacity, cold-cache experiment, importer backpressure, projection canary.

## Examples used

None.

## Cross-references

[Chapter 223 — Performance Mental Model](../../volumes/15-performance/223-performance-mental-model.md); [Chapter 226 — Profiling](../../volumes/15-performance/226-profiling.md); [Chapter 235 — Scaling](../../volumes/15-performance/235-scaling.md); [Chapter 256 — PHP-FPM](../../volumes/17-production-engineering/256-php-fpm.md); [Chapter 292 — Performance Investigation](../../volumes/20-senior-engineering/292-performance-investigation.md); [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md).

## Open threads

Continue with Chapter 307 — Security Checklist.

## Exact next section

Chapter 307 — Security Checklist: the Why This Matters section.

## Technical verification notes

The source contains a performance checklist and prose with no executable PHP blocks. Local Markdown links resolved and `git diff --check` passed. The chapter received a local editorial check for workload definition, percentiles, resource attribution, capacity, backpressure, correctness/security gates, rollout, and the Chapter 307 handoff. Live profiling, load, database, cache, queue, and deployment integrations were not run.

## Writing notes

Keep this summary short and update it after every writing session.
