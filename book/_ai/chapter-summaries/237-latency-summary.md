# AI Summary — Chapter 237 — Latency

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Defines latency as end-to-end work and waiting. Covers serial and parallel critical paths, distributions and percentiles, queueing and fan-out, monotonic PHP timing with hrtime, latency budgets, cold/warm paths, bounded parallelism, measurement, testing, and common optimization traps.

## Concepts already explained

Latency distribution, tail latency, critical path, fan-out, latency budget, cold path, warm path, and bounded parallelism.

## Terminology established

Serial work, parent budget, response margin, percentile population, dependency branch, and branch degradation.

## Examples used

Serial and parallel request diagrams, percentile composition intuition, an hrtime measurement helper, a latency-budget table, and a bounded fan-out policy.

## Cross-references

Chapter 224 — Measuring Performance, Chapter 238 on timeouts, and Volume XV performance chapters.

## Open threads

Continue with deadline and phase timeout design in Chapter 238.

## Exact next section

Chapter 238 — Timeouts: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP example. Local Markdown links resolved and git diff --check passed. No live latency or load test was run.

## Writing notes

Uses measurement-first performance language and treats queueing and dependency waits as first-class latency.
