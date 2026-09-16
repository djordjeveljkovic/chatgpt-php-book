# AI Summary — Chapter 231 — OPcache

- Status: complete
- Volume: Volume 15 — PERFORMANCE
- Last updated: 2026-09-16

## Written material

Explains compiled-script reuse, shared OPcache memory, configuration, timestamp validation, deployment invalidation, memory limits, preloading, JIT, status inspection, testing, and operations.

## Concepts already explained

OPcache reduces repeated parse/compile work but does not cache request data or fix slow queries. FPM and CLI may use different configuration and caches; timestamp policy must match deployment invalidation.

## Terminology established

OPcache, opcode, shared memory, timestamp validation, cache invalidation, preloading, JIT, cache warmup.

## Examples used

Production INI settings, atomic-release invalidation guidance, and a protected `opcache_get_status()` diagnostic.

## Cross-references

- [Chapter 059 — OPcache Internals](../../volumes/04-php-under-the-hood/059-opcache.md)
- [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 232 — PHP-FPM: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XV proofread. OPcache guidance links to the PHP Manual.
