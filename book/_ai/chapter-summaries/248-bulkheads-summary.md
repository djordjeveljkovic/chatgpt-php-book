# AI Summary — Chapter 248 — Bulkheads

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Explains bulkheads as isolation of in-flight work and resource budgets. Covers pool selection, semaphores, sizing across replicas, queues, tenants, PHP-FPM, circuit-breaker interaction, testing, and security. Includes a process-local Semaphore example.

## Concepts already explained

Bulkhead, resource pool, logical semaphore, physical isolation, pool fragmentation, tenant fairness, downstream budget, and noisy neighbor.

## Terminology established

Protected pool, consumer pool, local concurrency limit, global concurrency limit, idle fragmentation, and override path.

## Examples used

Shared-pool and bulkhead diagrams, typed Semaphore, pool-sizing equations, queue isolation, tenant tiers, and failure tests.

## Cross-references

Chapters 232, 246, and 247.

## Open threads

Continue with lease safety and fencing in Chapter 249.

## Exact next section

Chapter 249 — Distributed Locks: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP example. Local Markdown links resolved and git diff --check passed. No cross-process or load test was run.

## Writing notes

Separates concurrency isolation from circuit failure detection and from authorization.
