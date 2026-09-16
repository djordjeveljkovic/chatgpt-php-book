# AI Summary — Chapter 62 — PHP-FPM

- Status: complete
- Volume: Volume 5 — PHP RUNTIME
- Last updated: 2026-09-14

## Written material

Explains PHP-FPM master/pool/worker architecture, pool isolation, `static`/`dynamic`/`ondemand` modes, `pm.max_children`, `pm.max_requests`, memory-based sizing, timeout budgets, sockets, slow logs, status/ping protection, graceful reloads, deployment, overload, OOM, security, performance, and testing.

## Concepts already explained

FPM pool, master process, worker concurrency, process recycling, downstream budget, layered timeout, queue saturation, high-water memory, graceful reload, socket ownership, and status/slow-request observability.

## Terminology established

Finite worker pool, pool exhaustion, controlled damage, capacity ceiling, high-water memory, drain/reload, and smallest-bottleneck capacity.

## Examples used

Memory-budget arithmetic, illustrative pool configuration, timeout hierarchy, overload diagnosis, deployment compatibility, and load-test/measurement loops.

## Cross-references

Follows Chapter 61 and prepares for request lifecycle and worker processes in Chapters 63–64; connects to source/cache material in Chapters 40 and 59. References PHP Manual FPM installation/configuration/status and ini pages.

## Open threads

Request startup/teardown and response commitment are detailed in Chapter 63; process-level concurrency is detailed in Chapter 64.

## Exact next section

None — chapter complete.

## Technical verification notes

FPM directive names and modes are linked to the PHP Manual configuration reference. Numeric configuration values are explicitly illustrative and must be validated against the deployed PHP build.

## Writing notes

Never size `pm.max_children` without measured memory, downstream limits, and headroom.
