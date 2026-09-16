---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 50
title: PHP Arrays Internally
slug: php-arrays-internally
status: complete
summary: ../../_ai/chapter-summaries/050-php-arrays-internally-summary.md
---

# Chapter 50 — PHP Arrays Internally

## Why This Matters

PHP calls one type `array`, but that type serves several roles: list, map, ordered set-like collection, record, stack, queue, and sometimes an accidental object substitute. Internally, these values are HashTables. That gives PHP arrays flexible keys and stable insertion-order iteration, but it also gives them substantially more metadata than a compact language-level list in many other runtimes.

The practical questions are therefore not “are PHP arrays fast?” but:

- Is this value a list or a map?
- Do I need random key lookup or sequential iteration?
- How many elements and nested values will exist at once?
- Is the authoritative operation better in SQL?
- Will this structure be copied, serialized, cached, or held by every worker?

The engine’s packed representation can make list-shaped arrays more efficient, but it does not turn them into zero-overhead C vectors. The public array semantics remain the contract; the internal mode is a version-sensitive optimization.

## Mental Model

```text
PHP array zval
      │
      ▼
zend_array / HashTable
      ├── packed buckets for list-shaped keys
      └── hash-indexed buckets for general integer/string keys
              │
              ├── key (when string-keyed)
              └── zval value
```

The same PHP type can move between shapes:

```text
[] → packed list → hole/mixed key → general HashTable
```

The arrows describe a conceptual optimization transition. They do not promise that every mutation causes an immediate one-way conversion, nor that a particular PHP release makes exactly the same choice.

## Core Concept

A PHP array is an ordered map whose keys are integers or strings after PHP’s key-conversion rules. A list is a special case: keys are exactly `0, 1, ..., n - 1`. PHP 8.1 introduced `array_is_list()` as a language-level way to ask whether an array has that shape; using the function is safer than inferring internal packed status.

```php
var_dump(array_is_list(['a', 'b'])); // true
var_dump(array_is_list([1 => 'a', 2 => 'b'])); // false
```

Conceptually, a packed list can locate the nth element by position, while a general map needs key lookup. Both still store zval values and support PHP array behavior. The engine may optimize one path, but it must preserve observable keys, order, references, iteration rules, and copy-on-write semantics.

## How It Works

### List construction and append

```php
$queue = [];
$queue[] = 'first';
$queue[] = 'second';
$queue[] = 'third';
```

This creates consecutive integer keys and is a natural packed-list workload. Appending is generally amortized O(1), with occasional growth work. The values themselves are zvals, and strings/objects/arrays referenced by those zvals have their own lifetimes.

### Maps and records

```php
$reservation = [
    'court_id' => 7,
    'starts_at' => '2026-09-14T10:00:00+02:00',
    'ends_at' => '2026-09-14T11:00:00+02:00',
];
```

This is a map/record shape. Each string key participates in hashing and storage. It is convenient for boundaries such as decoded JSON, but a high-volume domain model may benefit from typed objects or a database row rather than thousands of nested associative arrays. The correct choice depends on lifecycle, validation, serialization, and access patterns—not on a universal rule that arrays are bad.

### Holes and reindexing

Deleting from a list creates a hole:

```php
$items = ['a', 'b', 'c'];
unset($items[1]);

var_dump($items); // keys 0 and 2 remain
var_dump(array_is_list($items)); // false
```

`array_values()` creates a new reindexed array:

```php
$items = array_values($items);
var_dump($items); // keys 0 and 1
```

That is an O(n) operation and can temporarily require memory for both representations. If the workload repeatedly deletes from the front, `array_shift()` also has costs that may make `SplQueue`, a database queue, or a different algorithm more appropriate. Measure the operation under its real data size.

### Key conversion

PHP converts some keys when an array is written. For example, an integer-looking string such as `'8'` can become integer key `8`, while a string with a leading zero such as `'08'` is treated differently. Boolean, float, and `null` keys also have documented conversions.

```php
$values = [];
$values['8'] = 'from string';
$values[8] = 'from integer';

var_dump($values); // the second assignment can replace the first
```

For external IDs, use a canonical representation that cannot collide with internal integer keys, or store the data in a typed value/object and validate at the boundary.

### Copy-on-write and nested arrays

```php
$original = [
    'meta' => ['state' => 'open'],
];
$copy = $original;

$copy['meta']['state'] = 'closed';

var_dump($original['meta']['state']); // open
var_dump($copy['meta']['state']); // closed
```

The engine can share array payloads while they are read-only and separate the mutated path as required by PHP value semantics. Nested structures may have their own sharing relationships. A shallow assignment is not the same as recursive serialization or a userland deep clone, and references can change the picture. Chapter 52 develops this in detail.

### Iteration and mutation

`foreach` observes PHP’s array iteration semantics, including key/value behavior and references:

```php
$values = ['a', 'b'];
foreach ($values as $key => $value) {
    echo $key, ':', $value, PHP_EOL;
}
```

A by-value loop does not make the whole array independent merely because it iterates. A by-reference loop creates aliases and must be cleaned up carefully:

```php
foreach ($values as &$value) {
    $value = strtoupper($value);
}
unset($value); // remove the lingering reference variable
```

The `unset()` is a practical guard against accidentally reusing `$value` as an alias later. Mutation during iteration has nuanced behavior; test the exact pattern rather than relying on a mental model from another language.

### Serialization and transport

Arrays are easy to pass to JSON, but PHP keys and JSON object/array rules are not identical:

```php
echo json_encode([0 => 'a', 2 => 'c'], JSON_THROW_ON_ERROR);
```

A non-consecutive integer-key array may encode as a JSON object rather than the JSON array shape a client expects. Normalize list data with `array_values()` when the wire contract requires a JSON array, and test the encoded schema. Serialization also walks values and can materialize or copy data, so it belongs in memory and latency measurements.

## What PHP Does

PHP documents arrays as ordered maps and defines key casting, iteration, append, deletion, sorting, references, and serialization behavior. These semantics apply regardless of whether Zend currently uses packed or general HashTable storage.

Use explicit API choices to communicate intent:

```php
function idsForResponse(array $ids): array
{
    return array_values($ids); // make list keys explicit for a JSON boundary
}
```

The return type says “array,” not “packed HashTable.” The implementation can change without changing the function’s PHP-level contract.

## What Zend Does

In current php-src, `zend_array` is built around `HashTable`. The PHP 8.4 definitions and operations can be studied in [Zend/zend_types.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h), [Zend/zend_hash.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.h), and [Zend/zend_hash.c](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.c). The source distinguishes packed/list-shaped and hash-indexed modes and provides specialized operations for common array paths.

Bucket layout, table masks, allocation sizes, packed-mode flags, deletion markers, and conversion heuristics are implementation-specific and can change between PHP releases. Avoid publishing exact byte figures unless they are measured and clearly tied to a PHP version, architecture, allocator, and build configuration.

## Minimal Example

```php
<?php

$events = [];
$events[] = ['type' => 'created', 'id' => 10];
$events[] = ['type' => 'confirmed', 'id' => 10];

foreach ($events as $event) {
    printf("%s #%d\n", $event['type'], $event['id']);
}
```

This is a packed outer list containing two associative inner maps. A useful memory diagram is:

```text
outer array/list
├── zval → inner map { type → string, id → int }
└── zval → inner map { type → string, id → int }
```

The nested maps and strings have independent value/lifetime costs. “Only two rows” does not mean “only two scalar slots.”

## Practical Example: Reservation Availability

Suppose an availability response is assembled from reservations:

```php
$byCourt = [];
foreach ($reservations as $reservation) {
    $courtId = $reservation['court_id'];
    $byCourt[$courtId][] = $reservation;
}
```

This gives average O(1) lookup by court ID and ordered lists within each court. It is a reasonable in-memory structure for a bounded result set. It becomes risky when the query returns an unbounded history, when every FPM worker builds the same structure, or when stale availability can cause a booking race. The database should own filtering, range checks, uniqueness, and concurrency constraints; PHP can shape the bounded result for a response.

```text
database: filter/index/range/conflict/constraint
    → PHP: map rows by court and format response
    → client: receive explicit list/map JSON contract
```

### Comparing representations

For a large import, compare:

```text
array of associative rows  → convenient, high structural overhead
array of typed objects      → identity/behavior, object overhead
generator/stream            → bounded live memory, one-pass constraints
database cursor/batching    → database owns storage, network/transaction trade-offs
```

Do not choose from folklore. Measure peak memory, throughput, failure recovery, and code clarity at the scale and concurrency of the actual service.

## Bad Example

```php
// Materialize every row, then scan it repeatedly for each request.
$allReservations = $repository->all();

foreach ($requestedCourts as $courtId) {
    $matches = array_filter(
        $allReservations,
        static fn (array $row): bool => $row['court_id'] === $courtId,
    );
}
```

This can be O(r × c) for r requested courts and c reservations, plus the memory cost of materializing everything. It may also use stale data and provide no concurrency guarantee.

## Better Example

Push authoritative filtering to the database, then build a bounded response map:

```php
$rows = $repository->findForCourtsAndWindow(
    courtIds: $requestedCourts,
    startsAt: $windowStart,
    endsAt: $windowEnd,
);

$byCourt = [];
foreach ($rows as $row) {
    $byCourt[$row['court_id']][] = $row;
}
```

The database query should have appropriate indexes and transaction/locking design. The PHP array is a presentation/data-shaping structure, not the source of truth for preventing overlapping reservations.

## Edge Cases

- A list with a hole is not a list-shaped array, even if it contains only integer keys.
- `array_values()` reindexes by creating a new result; account for time and peak memory.
- Numeric-string, float, boolean, and `null` keys can normalize unexpectedly.
- Nested arrays can share payloads until mutation and can contain references or cycles.
- A `foreach` reference can remain attached after the loop unless explicitly unset.
- JSON distinguishes list-like arrays and object-like maps; key shape affects the wire result.
- `array_shift()` and repeated front deletion can be expensive for large collections.
- Packed/general representation and exact bucket memory are version-sensitive.

## Performance

For a PHP array, total cost includes:

```text
outer HashTable + buckets + key storage
    + zval values + nested arrays/objects/strings
    + copy-on-write separations + serialization + allocator overhead
```

Appending to a list is typically amortized O(1); keyed lookup is O(1) average; scanning or filtering is O(n); sorting is generally O(n log n), subject to the exact function and comparison behavior. Reindexing, copying, JSON encoding, and nested traversal can all be O(n) in the number of reachable values.

At production scale, multiply per-request memory by possible worker count. A 30 MB result held by 20 workers has a different capacity impact from one 30 MB CLI process. Use pagination, streaming, SQL aggregation, and bounded batches when the full collection is not required.

## Security

Arrays often hold untrusted request data, permissions, decoded JSON, and secrets. Validate shape and types before indexing; do not assume a present key has the intended type. Avoid logging complete arrays that contain tokens or personal data. Bound nesting depth, element count, and payload size when accepting arbitrary JSON to reduce memory-exhaustion and parser/resource risks.

For authorization, an array of roles is only input to a policy decision. Use canonical identifiers and explicit allowlists; do not let PHP key conversion or loose comparisons decide access.

## Testing

1. Test list/map shape with `array_is_list()` and JSON encoding expectations.
2. Test key casting for external identifiers and collision/overwrite cases.
3. Test copy-on-write behavior and by-reference iteration cleanup at the language level.
4. Test large, nested, sparse, deletion-heavy, and serialization workloads in separate processes.
5. Test database-backed availability under concurrent requests; a PHP array cannot replace a database constraint.

For performance fixtures, record PHP version, architecture, OPcache/JIT state, memory limit, input cardinality, and peak memory. Repeat enough times to separate startup and allocator noise from the operation being measured.

## Common Mistakes

- Treating every PHP array as a compact list.
- Calling an array “a hash map” while ignoring insertion order and packed mode.
- Assuming append, front deletion, scan, and keyed lookup have the same complexity.
- Using an in-memory array as the concurrency authority for a database-backed invariant.
- Forgetting that nested arrays multiply zval and metadata costs.
- Assuming `array_values()` is a free view instead of a new result.
- Returning sparse integer keys to JSON without testing the wire shape.

## Senior Engineer Thinking

An array decision should be explicit:

```text
required semantics: list / map / record / queue / set-like membership
    → key and iteration contract
    → cardinality and mutation pattern
    → memory per worker and lifetime
    → database/cache/stream boundary
    → serialization and security contract
    → measured implementation choice
```

The most important optimization is often reducing the number of values materialized, not finding a clever array expression. A packed list can help, but paging 10 million rows is usually a larger win than rearranging a 10-million-element array after it has already been built.

## Exercises

1. Write examples of a packed list, sparse list, map, and nested record. Use `array_is_list()` and predict JSON output.
2. Measure the time and peak memory of `array_values()` on a large sparse array.
3. Compare repeated `array_filter()` scans with one map grouped by `court_id`; state the complexity and memory trade-off.
4. Build a fixture that demonstrates a lingering `foreach` reference and fix it with `unset()`.
5. Read the PHP-8.4 HashTable source and document which observations are public array semantics and which are private packed/bucket implementation details.

## Review Questions

1. Why can one PHP array type represent both a list and a map?
2. What conditions make an array list-shaped, and how can userland test that?
3. Why can removing an element and calling `array_values()` create a memory spike?
4. Why should a PHP array not be the authority for a concurrent reservation invariant?
5. Which array costs should be included in capacity planning for PHP-FPM workers?

## Summary

PHP arrays are ordered maps implemented in current Zend versions around HashTables, with a packed optimization for some consecutive integer-key shapes. Lists, maps, records, sparse collections, and nested data therefore share flexible semantics but not identical costs. Key casting, copy-on-write, iteration references, serialization, allocation, and worker memory all matter. Use arrays for bounded, appropriate workloads; keep authoritative filtering and concurrency constraints in the database or another suitable boundary, and treat packed/bucket layouts as version-sensitive internals.

## References

- [PHP Manual: arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: `array_is_list`](https://www.php.net/manual/en/function.array-is-list.php)
- [PHP Manual: `foreach`](https://www.php.net/manual/en/control-structures.foreach.php)
- [PHP Manual: JSON encoding](https://www.php.net/manual/en/function.json-encode.php)
- [php-src PHP-8.4: `Zend/zend_types.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h)
- [php-src PHP-8.4: `Zend/zend_hash.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.h)
- [php-src PHP-8.4: `Zend/zend_hash.c`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.c)
