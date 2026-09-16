# AI Summary — Chapter 79 — Sorting

- Status: complete
- Volume: Volume 6 — ALGORITHMS AND DATA STRUCTURES
- Last updated: 2026-09-15

## Written material

Complete chapter draft explains ordering contracts, stability versus determinism, built-in sort comparison flags, key-preserving and reindexing PHP APIs, comparator correctness, multi-column sorting, precomputed sort keys, database ordering and pagination, streaming limits, complexity and memory, security, testing, common mistakes, exercises, review questions, and summary. Editorial proofread is pending.

## Concepts already explained

Ordering contract, comparison policy, stability, deterministic order, tie-breaker, comparator laws, key preservation, lexicographic multi-column sorting, decorate-sort-undecorate, database-side ordering, global sort versus streaming, and bounded top-k selection.

## Terminology established

Stable sort preserves relative order of equal elements; deterministic sorting requires a defined complete order and defined input behavior. `sort()`/`usort()` reindex values; `asort()`/`uasort()` preserve key/value association; key sorting is separate. A custom comparator returns an integer sign and must be pure and consistent.

## Examples used

Numeric and natural-order built-in flags; `sort()` versus `asort()` keys; a composite `usort()`/`uasort()` task comparator; `array_multisort()` with parallel columns; decorate-sort-undecorate; SQL ordering before pagination.

## Cross-references

Chapters 52, 74–75, 83–84, and 90 for copy-on-write, memory, arrays, heaps, priority queues, and streaming; Chapters 104, 108–110, and 119 for SQL ordering, indexes, query plans, and pagination; Chapter 144 for SQL injection.

## Open threads

Chapter 80 — Searching is complete; Chapter 81 — Binary Search follows.

## Exact next section

Chapter complete; Chapter 81 — Binary Search: the Why This Matters section.

## Technical verification notes

Checked the official PHP Manual pages for `sort()`, the sorting overview, `usort()`, `uasort()`, and `array_multisort()`. Verified stable ordering since PHP 8.0, key preservation/reindexing behavior, comparator integer-sign semantics and non-integer cast behavior, sort flags, and multi-array lexicographic sorting. Complexity and implementation details are explicitly qualified because the Manual does not promise a portable algorithm or complexity bound.

## Writing notes

Decorate-sort-undecorate records with an explicit input sequence number when stable input-relative tie ordering is required. Keep SQL identifiers allowlisted, use a unique final key for deterministic pagination, and distinguish stable ties from a complete order.
