# AI Summary — Chapter 56 — Memory Manager

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Explains PHP-managed memory versus process RSS and OS/container limits; request and persistent allocation lifetimes; Zend MM conceptually; allocator reuse and fragmentation; `memory_get_usage()`, peak usage, `memory_limit`, and `gc_mem_caches()`; array overhead; copy-on-write; streaming and bounded batches; PHP-FPM capacity planning; native memory; security, testing, exercises, and review questions.

## Concepts already explained

Allocation lifetime, request memory, persistent memory, Zend MM, allocator-reserved memory, RSS, peak memory, bounded data flow, copy-on-write cost, worker capacity, native allocations, memory exhaustion.

## Terminology established

Zend memory manager, request-lived allocation, persistent allocation, PHP-managed usage, real allocated memory, resident set size, allocator reuse, fragmentation, streaming, batch boundary.

## Examples used

Memory instrumentation; whole-file versus streaming input; copy-on-write assignment; PHP-FPM worker sizing; an unbounded exporter; bounded batch exporting; by-reference loop and native-memory edge cases.

## Cross-references

Links to Chapters 47, 49, 50, 51, and 52 for zvals, HashTables, arrays, references, and copy-on-write. Connects worker capacity to Volume V.

## Open threads

Later runtime chapters should apply the distinction between PHP-managed memory, worker RSS, shared OPcache memory, and OS/container limits.

## Exact next section

Chapter 57 — Garbage Collection: the Why This Matters section.

## Technical verification notes

Uses official PHP documentation for memory functions and `memory_limit`, plus PHP source links for Zend allocator code. Exact allocator bins, thresholds, and accounting are explicitly treated as implementation/build-sensitive rather than fixed application contracts.
