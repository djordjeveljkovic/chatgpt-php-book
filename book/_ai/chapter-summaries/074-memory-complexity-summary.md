# AI Summary — Chapter 74 — Memory Complexity

- Status: complete
- Volume: Volume 6 — ALGORITHMS AND DATA STRUCTURES
- Last updated: 2026-09-14

## Written material

Explains input, auxiliary, output, and peak live space; hidden duplication and copy-on-write; recursion; PHP array overhead; generators; whole-file versus streaming and bounded batches; memory measurement, worker lifetime, security, testing, exercises, review questions, and summary.

## Concepts already explained

Memory complexity, auxiliary space, peak live memory, live range, COW separation, materialization, streaming, bounded batch, PHP-managed memory, RSS, allocator reuse, and worker retention.

## Terminology established

Whole-file transformation, streaming uppercase output, NDJSON line generator, bounded 500-item batches, recursive depth, and overlapping array transformations.

## Examples used

Connects to Chapters 47–52 and 56–57 for zvals, COW, memory management, and garbage collection; prepares for queues, streaming algorithms, workers, and production capacity planning.

## Cross-references

Later chapters should apply bounded memory to queues, batching, memoization, streaming, and production worker limits.

## Open threads

The next chapter applies memory and access trade-offs to PHP arrays and hash maps.

## Exact next section

Chapter complete; Chapter 75 — Arrays and Hash Maps follows.

## Technical verification notes

Uses the PHP Manual for memory functions, `memory_limit`, generators, and arrays. It distinguishes PHP-managed allocation from RSS/native memory and treats exact per-entry costs as workload/build-sensitive.

## Writing notes

Peak live memory and worker lifetime are emphasized over final result size alone.
