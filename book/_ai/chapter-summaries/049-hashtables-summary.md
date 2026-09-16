# AI Summary — Chapter 49 — HashTables

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 49 explains HashTables as ordered key/value structures used by PHP arrays and other engine data. It covers hashing, collision resolution, expected complexity, insertion order, growth, deletion/rebuild effects, packed mode, PHP key casting, local-map versus database-index decisions, performance, security, testing, exercises, and review questions.

## Concepts already explained

HashTable, bucket, hash/index position, collision, insertion order, packed mode, general hash mode, average O(1), resize/rebuild, deletion capacity, key normalization.

## Terminology established

Hash-to-bucket diagram; collision-chain model; packed/general mode distinction; map-building complexity; local map versus authoritative database boundary.

## Examples used

Ordered map iteration, deletion from a large map, packed-to-mixed shape, membership map versus `in_array()`, and canonical external keys.

## Cross-references

Builds on Chapters 46–48 and points to Chapter 50 for PHP arrays. It references Chapter 48 for cached string hashes and uses the PHP arrays/array-functions manuals plus PHP-8.4 `zend_hash.h/.c` and `zend_types.h`.

## Open threads

Chapter 50 applies HashTables to PHP array list/map shapes; later chapters revisit memory and copy-on-write implications.

## Exact next section

None — chapter complete.

## Technical verification notes

Complexity statements are qualified as expected/average and include resize costs. Key casting and iteration claims follow the PHP array manual. Bucket/index layouts, masks, packed flags, collision links, and deletion internals are explicitly marked PHP-8.4/source-version-sensitive.

## Writing notes

Do not confuse HashTable slot order with PHP insertion order, or a local fast map with an authoritative shared data structure. Keep hash functions separate from password/security primitives.
