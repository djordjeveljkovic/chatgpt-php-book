---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 49
title: HashTables
slug: hashtables
status: complete
summary: ../../_ai/chapter-summaries/049-hashtables-summary.md
---

# Chapter 49 — HashTables

## Why This Matters

Hash tables are the engine’s general-purpose mapping structure. PHP arrays use them, but so do many internal symbol tables, property tables, caches, and extension data structures. When a PHP operation looks like “find the value for this key,” a HashTable is often somewhere underneath.

This matters because a HashTable combines several properties that applications often need:

- average constant-time lookup by key;
- insertion-order iteration;
- integer and string keys;
- dynamic growth and deletion;
- a packed representation for some list-shaped data.

Those properties have trade-offs. Hash tables consume more metadata than a compact C array, collisions and resizing exist, and PHP’s flexible key rules can change the key you thought you inserted. Chapter 50 applies this structure specifically to PHP arrays; this chapter builds the general model first.

## Mental Model

```text
key ──hash──► index table ──► bucket position ──► bucket
                                                   ├── key
                                                   └── zval value

bucket storage also preserves insertion order for iteration
```

A simplified insertion flow is:

```text
normalize key according to the owning API
    → compute/find hash position
    → follow collision chain or empty slot
    → insert/update bucket
    → grow/rebuild if capacity policy requires it
```

The actual PHP HashTable has packed and hash modes, compact memory layouts, bit masks, tombstones, and version-specific macros. The diagram is a reasoning model, not a layout contract.

## Core Concept

A hash table stores key/value associations in buckets. A hash function maps a key to an index, but different keys can map to the same index. The implementation resolves collisions while preserving the ability to find the correct key.

For expected lookup complexity:

```text
lookup / insert / delete: O(1) average
worst case:              O(n) in a degraded collision scenario
iteration:               O(n)
resize/rebuild:          O(n), amortized over growth
```

The average claim assumes a healthy hash function, capacity policy, and workload. It is not a promise of one CPU instruction or of constant memory use.

## How It Works

### Keys and hashes

For a string key, the table hashes the bytes and uses the result to locate a candidate bucket. It still compares the key, because a hash collision does not mean equal keys. For an integer key, the implementation can use the integer directly in its hash-table path. A cached string hash can avoid repeated hashing in some cases (Chapter 48).

Conceptually:

```text
status → hash(status) → slot 5
state  → hash(state)  → slot 5

slot 5 → bucket for status → next bucket for state
```

The collision-chain drawing is conceptual. Current `HashTable` internals encode links and positions in compact, version-specific fields.

### Insertion order

PHP’s array iteration order is observable:

```php
$values = [];
$values['first'] = 1;
$values['second'] = 2;
$values['third'] = 3;

foreach ($values as $key => $value) {
    echo $key, '=', $value, PHP_EOL;
}
```

The output follows insertion order. Updating an existing key changes its value but does not generally move it to the end; removing and re-inserting is a different operation. A hash table implementation therefore needs both efficient lookup and an iteration sequence.

### Growth and deletion

As entries grow, the table allocates a larger table and repositions entries. This is an O(n) event, but a geometric growth policy spreads the cost across many operations. A deletion leaves a state that allows later searches and iteration to remain correct; the implementation may use tombstones or compact/rebuild storage depending on mode and version.

```php
$map = [];
for ($i = 0; $i < 100_000; ++$i) {
    $map['key-' . $i] = $i;
}

unset($map['key-50000']);
```

The logical entry is gone. The memory footprint need not immediately return to the minimum possible size. If this pattern repeats in a long-running worker, measure capacity and peak memory rather than assuming `unset()` means the allocator returns every byte to the operating system.

### Packed mode

When keys form a list-like sequence of non-negative integers, PHP can use a packed HashTable representation. It avoids some general string-key/hash-index work while retaining array semantics. Adding a non-list key or creating a hole can require a transition to general hash mode; the exact transition rules are implementation-specific.

```php
$list = [10, 20, 30];       // list-shaped
$list[] = 40;
$list['name'] = 'Ada';      // mixed keys; general map behavior is needed
```

“Packed” does not mean a PHP array becomes a C `int[]` with no zvals or metadata. It is an engine optimization within HashTable semantics. Chapter 50 explores the memory and API consequences.

## What PHP Does

PHP defines key conversion and iteration behavior. The manual documents that array keys can be integers or strings and that some supplied key values are converted: for example, integral numeric strings without a leading plus can become integer keys, while strings with leading zeros are treated differently. Booleans, floats, and `null` also have documented conversions.

```php
$values = [];
$values['8'] = 'string-looking key';
$values[8] = 'integer key';

var_dump($values); // one logical integer key can be overwritten
```

Check the [array key casting rules](https://www.php.net/manual/en/language.types.array.php) rather than designing a protocol around intuition. If external identifiers must remain strings, normalize and validate them explicitly.

## What Zend Does

The PHP 8.4 HashTable API and implementation are represented by [Zend/zend_hash.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.h) and [Zend/zend_hash.c](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.c). The headers expose engine-facing operations and macros for initialization, lookup, insertion, deletion, iteration, and packed/hash checks. The structure and bucket fields are private implementation details even when an extension includes the header.

Current source uses a `HashTable` with `Bucket` storage and a table/index arrangement optimized for PHP’s workload. Flags distinguish modes and states; capacity, masks, and internal offsets are not stable across releases or architectures. Pin the PHP source tag and binary when reading exact fields.

## Minimal Example

```php
<?php

$inventory = [
    'racket' => 4,
    'balls' => 12,
];

if (array_key_exists('racket', $inventory)) {
    $inventory['racket'] -= 1;
}

foreach ($inventory as $item => $quantity) {
    printf('%s: %d\n', $item, $quantity);
}
```

The application sees a map with ordered iteration and average constant-time key lookup. The implementation may use a general hash table, while the application should focus on the invariant that quantities are updated under the intended key.

## Practical Example: Choosing a Lookup Structure

Suppose a request repeatedly asks whether 50,000 user IDs are allowed:

```php
$allowed = [];
foreach ($allowedIds as $id) {
    $allowed[$id] = true;
}

foreach ($requestedIds as $id) {
    if (isset($allowed[$id])) {
        // O(1) average lookup in the in-memory map.
    }
}
```

Building the map is O(n) time and space; each membership check is O(1) average. A repeated `in_array($id, $allowedIds, true)` scan is O(n) per request ID, which can become O(nm) for n stored IDs and m queries. But the database may be the better owner if the allowed set is authoritative or too large to materialize in every worker. Algorithm choice includes data ownership and network cost, not only big-O notation.

## Bad Example

```php
// Assuming every externally supplied numeric-looking key remains a string.
$byExternalId[(string) $input] = $record;
```

If the external format permits values such as `8`, PHP’s array-key conversion can make it an integer key. That may collide with a later integer lookup or with a different normalization path.

## Better Example

Choose a canonical key representation and test it:

```php
function externalKey(string $raw): string
{
    if (!preg_match('/\Ausr_[0-9]+\z/D', $raw)) {
        throw new InvalidArgumentException('Invalid external ID');
    }

    return $raw;
}

$records[externalKey($input)] = $record;
```

The prefix makes the domain key unambiguously string-shaped. This is a domain decision, not a trick to expose the HashTable.

## Edge Cases

- Equal hash values still require key equality checks.
- Deleting entries may leave capacity or tombstone effects until a rebuild.
- Iteration order is a PHP-visible property; hash-table slot order is not necessarily the iteration order.
- Mutating a table while iterating has documented language behavior and can involve internal iterator state; test the exact operation.
- Recursive arrays can create cycles that affect traversal, serialization, and debugging.
- Integer/string key conversion can cause accidental overwrites.
- Hashing is for table placement, not password storage or collision-resistant application identifiers.
- A packed table can become a general map when key shape changes; do not assume one representation for the lifetime of an array.

## Performance

Hash-table performance depends on:

```text
lookup cost + key hashing/comparison + bucket/value metadata
             + allocation + cache locality + resize/rebuild cost
```

Use maps for repeated membership or key lookup when the set fits the process memory budget. Do not build a 10-million-entry map in every FPM worker merely to avoid a database query without measuring total capacity and freshness requirements. A compact database index can be cheaper than PHP memory and network transfer.

Benchmark the operations you actually need: integer lookup, string lookup, iteration, deletion, and construction. Include realistic key lengths and hit/miss ratios.

## Security

Never trust an array key merely because HashTables are fast. Normalize untrusted identifiers before lookup, use strict comparisons where appropriate, and defend against resource exhaustion by bounding the number and size of keys. Hash-table collision behavior is an engine concern; do not expose internal hashes as an authentication or password mechanism.

When a key selects a file, SQL fragment, permission, or callable, the security decision must validate the value against an allowlist or structured API. A successful map lookup proves only that a key exists.

## Testing

1. Test integer, string, numeric-string, boolean, float, and `null` key inputs against the documented conversion rules.
2. Test insertion, update, delete, and re-insertion order.
3. Test empty, one-entry, collision-like, large, and deletion-heavy workloads.
4. Test memory and latency in a separate process with fixed PHP version/configuration.
5. Test authoritative database behavior when an in-memory map is used only as a cache.

For engine investigations, use a pinned php-src checkout and targeted C tests. Do not snapshot private bucket offsets as portable behavior.

## Common Mistakes

- Saying hash-table lookup is always O(1) without the average-case qualification.
- Confusing bucket/index order with PHP’s insertion order.
- Ignoring key casting and accidental overwrites.
- Assuming `unset()` immediately shrinks a table to its minimum footprint.
- Building a large per-worker map when the database or a shared cache owns the data.
- Treating packed mode as an ordinary C array with no PHP value overhead.
- Using a hash function as a security primitive.

## Senior Engineer Thinking

For a lookup requirement, write down:

```text
key domain and normalization
    → expected lookup/iteration pattern
    → count, key length, and mutation rate
    → memory and worker-capacity budget
    → freshness/ownership boundary
    → HashTable, database index, or shared-cache choice
```

The HashTable explains the cost of an in-process map. It does not automatically make that map the right system boundary. A fast local lookup can be wrong if it is stale, duplicated across workers, or too expensive to build.

## Exercises

1. Implement membership testing with a list scan and with a HashTable map. State the time and space complexity of each.
2. Write a fixture showing how PHP treats '8', '08', 8, `true`, and `null` as keys.
3. Measure iteration order after updates, deletions, and re-insertions.
4. Compare a 100,000-key local map with an indexed database existence query under realistic latency.
5. Read `zend_hash.h` in the PHP-8.4 branch and identify the public-looking API operations versus fields that are unsafe to treat as stable.

## Review Questions

1. Why does a hash table need both a hash position and a key comparison?
2. What is the difference between expected O(1) lookup and guaranteed O(1) lookup?
3. Why does insertion order require more than a bucket slot lookup?
4. What can happen to capacity after deletion?
5. When is a database index a better lookup structure than a PHP HashTable?

## Summary

A HashTable maps integer or string keys to zval values, resolves collisions, grows and rebuilds as needed, and preserves PHP-visible insertion order. Lookups are O(1) on average, not an unconditional guarantee. PHP also has a packed mode for some list-shaped arrays, but key conversion, memory overhead, deletion behavior, and worker capacity remain important. The `HashTable` and `Bucket` layouts are version-sensitive php-src details; choose the structure from workload, ownership, and consistency requirements.

## References

- [PHP Manual: arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: array functions](https://www.php.net/manual/en/ref.array.php)
- [php-src PHP-8.4: `Zend/zend_hash.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.h)
- [php-src PHP-8.4: `Zend/zend_hash.c`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.c)
- [php-src PHP-8.4: `Zend/zend_types.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h)
- [PHP Manual: `array_is_list`](https://www.php.net/manual/en/function.array-is-list.php)
