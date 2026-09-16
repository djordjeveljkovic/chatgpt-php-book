# AI Summary — Chapter 80 — Searching

- Status: complete
- Volume: Volume 6 — ALGORITHMS AND DATA STRUCTURES
- Last updated: 2026-09-15

## Written material

Complete chapter covers linear scans with `in_array()`, `array_search()`, custom predicates, key lookup, nullable values and presence, repeated-query indexes, sorting/search trade-offs, database boundaries, complexity/memory, production constraints, tests, exercises, review questions, and summary.

## Concepts already explained

Linear search, membership, key lookup, lookup index, equality policy, canonicalization, duplicate policy, presence versus null value, freshness, and search/data ownership boundary.

## Terminology established

Value search versus key search; first matching key; expected O(1) average map lookup; index build and refresh cost; hit/miss cases; sorted-search trade-off.

## Examples used

Strict role membership; first confirmed reservation scan; `array_search()` at key `0`; nullable cache-key presence; customer-by-ID map for order matching.

## Cross-references

Chapters 73 and 75 explain complexity and PHP maps; Chapter 79 covers sorting; Chapter 81 develops binary search; Chapters 108–111 cover database indexes and query plans; Chapters 64–65 explain process boundaries.

## Open threads

Binary search mechanics intentionally remain in Chapter 81.

## Exact next section

Chapter complete; Chapter 81 — Binary Search: the Why This Matters section.

## Technical verification notes

Checked PHP Manual behavior for `in_array()` loose/strict comparison, `array_search()` first matching key/`false` sentinel and strict mode, `array_key_exists()` versus `isset()` with null values, first-dimension-only key checks, and PHP 8.5 deprecation of null as an offset/key. Complexity language is model-based rather than a PHP Manual guarantee.

## Writing notes

Keep equality, duplicate semantics, index lifetime, and PHP/database ownership explicit. Binary search mechanics belong in Chapter 81.
