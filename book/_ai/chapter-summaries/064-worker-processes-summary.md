# AI Summary — Chapter 64 — Worker Processes

- Status: complete
- Volume: Volume 5 — PHP RUNTIME
- Last updated: 2026-09-14

## Written material

Explains PHP worker processes as concurrency, capacity, and failure-isolation units. Covers process/request/dependency concurrency, queueing, Little’s Law, bottleneck budgets, memory sizing, slow dependencies, retries, worker recycling, restart herds, shared-state security, metrics, load testing, and concurrency correctness.

## Concepts already explained

One-request-per-worker model, finite worker pool, queue growth, effective concurrency, downstream contention, worker reuse, high-water memory, partial failure isolation, retry amplification, and durable invariants.

## Terminology established

Worker concurrency budget, smallest bottleneck, queueing failure, retry storm, high-water mark, and process isolation versus tenant isolation.

## Examples used

Memory/capacity arithmetic, slow-upstream scenario, worker metrics, concurrent invariant test, retry storm diagram, and deployment/recycle behavior.

## Cross-references

Builds on FPM Chapter 62 and lifecycle Chapter 63, and leads to long-running process state in Chapter 65. References PHP FPM, status, PCNTL, OPcache, and memory manuals.

## Open threads

Long-running job/daemon lifecycle, signal draining, connection freshness, and persistent-memory hazards are developed in Chapter 65.

## Exact next section

None — chapter complete.

## Technical verification notes

Capacity equations are presented as approximations and operational reasoning tools. FPM and process-control claims are linked to PHP Manual references; actual worker behavior remains SAPI/configuration dependent.

## Writing notes

Do not treat a higher worker count as free parallelism; correlate PHP capacity with databases and external services.
