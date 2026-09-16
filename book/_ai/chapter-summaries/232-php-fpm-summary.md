# AI Summary — Chapter 232 — PHP-FPM

- Status: complete
- Volume: Volume 15 — PERFORMANCE
- Last updated: 2026-09-16

## Written material

Explains FPM as a bounded FastCGI worker pool, process modes, memory-based capacity, worker recycling, timeouts, listen queues, graceful reloads, readiness, observability, testing, and overload operations.

## Concepts already explained

FPM capacity is constrained by worker memory and downstream dependencies. More children can move rather than remove queueing; process liveness differs from readiness. Recycling mitigates impact from growth but does not fix leaks.

## Terminology established

PHP-FPM, worker pool, `pm.max_children`, process mode, listen backlog, request termination, graceful reload, slowlog, readiness.

## Examples used

A memory-based capacity equation, deployment/reload policy, timeout relationships, and FPM observability signals.

## Cross-references

- [Chapter 062 — PHP-FPM](../../volumes/05-php-runtime/062-php-fpm.md)
- [Chapter 231 — OPcache](../../volumes/15-performance/231-opcache.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 233 — Caching: the Why This Matters section.

## Technical verification notes

Local links were linted in the consolidated Volume XV proofread. FPM guidance links to the PHP Manual.
