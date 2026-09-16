# AI Summary — Chapter 59 — OPcache

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Explains source-to-opcode-to-execution flow; shared compiled script data versus process-local values; cache hits/misses; timestamp validation and explicit invalidation; CLI/FPM configuration; deployment with immutable releases; OPcache capacity and monitoring; preloading from PHP 7.4; JIT from PHP 8.0; security, testing, exercises, and review questions.

## Concepts already explained

Opcode cache, cache key, cache hit/miss, timestamp validation, revalidation frequency, explicit reset/reload, shared OPcache memory, immutable release, preload, JIT, cache churn, warmup.

## Terminology established

OPcache, compiled representation, invalidation protocol, worker drain, build marker, shared memory, timestamp validation, preloading, JIT.

## Examples used

Protected status/configuration inspection; deployment sequence; configuration decision table; build marker; immutable release layout; disabled-validation deployment contract; cold/warm verification.

## Cross-references

Connects OPcache to Chapters 40–45 on source, compilation, and opcodes, Chapter 58 on extensions, Chapter 56 on shared versus per-process memory, and Volume V on PHP-FPM/workers.

## Open threads

Later performance and production volumes should revisit cache capacity, worker reloads, preloading/JIT evidence, and deployment rollback.

## Exact next section

The next chapter after this batch is Chapter 60 — PHP CLI, beginning with its Why This Matters section.

## Technical verification notes

Uses official OPcache, configuration, status, preloading, and JIT manual references plus php-src `ext/opcache`. Defaults and exact cache-key/internal behavior are explicitly treated as target-build/configuration-sensitive.
