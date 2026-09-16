---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 75
title: Arrays and Hash Maps
slug: arrays-and-hash-maps
status: complete
summary: ../../_ai/chapter-summaries/075-arrays-and-hash-maps-summary.md
---

# Chapter 75 — Arrays and Hash Maps

## Why This Matters

PHP arrays are one of the language’s most useful and most misleading abstractions. The same syntax can represent a list, an associative map, a queue, a stack, a set-like collection, or a nested record. That convenience is productive, but it can blur the difference between operations and hide costs.

The PHP manual describes an array as an ordered map: a structure associating values with integer or string keys while preserving iteration order. It is not only a compact numeric vector. When a program handles large collections or performs repeated lookup, understanding the map model helps prevent accidental scans, key collisions, silent overwrites, and avoidable memory growth.

## Mental Model

```text
PHP array
├── key → value association
├── insertion order
├── integer and string keys
├── list-like append/index access
└── nested values, objects, and references
```

Think of three distinct uses:

```php
$list = ['red', 'green', 'blue'];

$map = ['us' => 'United States', 'ca' => 'Canada'];

$setLike = ['admin' => true, 'editor' => true];
```

All are arrays, but their invariants differ. A list cares about position and order. A map cares about key identity. A set cares only whether a key is present. Naming the use makes review and testing clearer.

## Core Concept

Repeated value search scans a list:

```php
$names = ['Ada', 'Grace', 'Edsger'];

if (in_array('Grace', $names, true)) {
    // O(n) value search
}
```

When the same names are queried repeatedly, index them by a canonical key:

```php
$personByName = [
    'Ada' => ['id' => 1],
    'Grace' => ['id' => 2],
    'Edsger' => ['id' => 3],
];

$person = $personByName['Grace'] ?? null;
```

The map gives direct key access in the usual average-case model, but it changes the data contract. The key must be unique, canonical, and safe to derive. If two records have the same name, a map of `name => record` must reject duplicates, choose a policy, or store a list of records per name.

## How It Works

### Keys are data, not decoration

PHP converts some keys according to its array-key rules. Integer-looking strings can become integer keys in cases defined by the language. Boolean, floating-point, and null keys are also converted when used as array keys. Do not assume that every textual input remains a distinct string key.

When external identifiers can look numeric, canonicalize them explicitly:

```php
function externalKey(string $raw): string
{
    $key = trim($raw);
    if ($key === '') {
        throw new InvalidArgumentException('Identifier cannot be empty.');
    }

    return 'id:' . $key;
}
```

The prefix is not a universal solution, but it makes the key domain explicit and avoids accidental collisions with unrelated internal keys. Better still, use a value object or a separate map when the domain has more structure than a string can safely carry.

### Presence is not truthiness

These checks answer different questions:

```php
$data = ['status' => null, 'count' => 0, 'enabled' => false];

array_key_exists('status', $data); // key exists, value may be null
isset($data['status']);            // false because value is null

array_key_exists('count', $data);  // true
isset($data['count']);             // true
```

Use `array_key_exists()` when presence matters, including nullable values. Use `isset()` when a missing or null value should have the same meaning. The null-coalescing operator follows an `isset()`-like existence check:

```php
$timeout = $config['timeout'] ?? 30;
```

Do not use `empty()` to test domain presence unless its loose rules are exactly what the contract requires. A legitimate value such as `0` or `'0'` can be treated as empty.

### Lists and keys

Appending is natural for a list:

```php
$queue = [];
$queue[] = 'job-1';
$queue[] = 'job-2';
```

But removing the first element with `array_shift()` can require moving or reindexing the remaining list and is a poor queue strategy for large collections. Chapter 78 will compare queue designs. If keys carry identity, `array_values()` may destroy information by reindexing. Preserve or discard keys intentionally.

### Hash-map intuition

A hash map uses a key to locate a bucket or entry rather than scanning every value. Collisions are handled by the implementation. Resizing, key hashing, string length, allocation, and value access all contribute to actual cost. PHP’s HashTable is an internal implementation detail explained in Chapter 49; application code should rely on language semantics and measured behavior, not bucket layout.

The average-case lookup model is useful for design, but it is not an SLA. A request can still be slow because it builds a large map, copies or separates it, decodes values, calls a database, or serializes the result.

## Practical Example: Building a Safe Index

Silently overwriting duplicate keys is dangerous:

```php
$userByEmail = [];

foreach ($users as $user) {
    $userByEmail[strtolower($user['email'])] = $user;
}
```

If two rows normalize to the same email, the later one wins without evidence. Make the policy explicit:

```php
/** @return array<string, array{id: int, email: string}> */
function indexUsersByEmail(iterable $users): array
{
    $index = [];

    foreach ($users as $user) {
        $email = strtolower(trim($user['email']));

        if (array_key_exists($email, $index)) {
            throw new LogicException('Duplicate email in source data.');
        }

        $index[$email] = $user;
    }

    return $index;
}
```

In a real system, avoid exposing the email in an exception if it is sensitive. The example emphasizes the invariant: one canonical email maps to one user.

### Grouping instead of overwriting

If duplicates are expected and all records matter, map each key to a list:

```php
$ordersByCustomer = [];

foreach ($orders as $order) {
    $customerId = $order['customer_id'];
    $ordersByCustomer[$customerId][] = $order;
}
```

This gives lookup by customer followed by iteration through that customer’s orders. The space is proportional to the input and the result preserves duplicates. Document whether order within each group matters.

## What PHP Does

PHP preserves insertion order for arrays. Updating an existing key does not mean the value is a new list item; it updates the key’s value. Unsetting a key leaves a gap in numeric keys until code explicitly reindexes. Numeric key behavior and insertion order make PHP arrays different from a pure abstract map and from a packed vector.

The array functions also differ in whether they preserve keys, compare strictly, mutate in place, or return a new array. Read the function contract before using it inside a hot path or a memory-sensitive pipeline. Functions that return a new array can temporarily keep both source and result alive.

For object keys, ordinary PHP arrays are not a general object-key map. `SplObjectStorage` provides an object-to-data map and can act as an object set. Use it when object identity—not a string representation—is the key.

## Performance

Choose based on dominant operations:

| Need | Natural representation | Main trade-off |
| --- | --- | --- |
| Ordered traversal | list-like array | value search is linear |
| Repeated string/integer lookup | associative array | key storage and canonicalization |
| Many records per key | map of lists | additional nesting and output size |
| Object identity lookup | `SplObjectStorage` | object-specific API |
| Bounded numeric storage | specialized structure or streaming | less general convenience |

The table is a starting point, not a benchmark result. Test with actual key lengths, value types, cardinality, and access patterns. Chapter 74’s memory discussion is especially important for large PHP arrays.

Avoid rebuilding the same index inside a loop:

```php
foreach ($batches as $batch) {
    $byId = buildIndex($catalog); // repeated work
    process($batch, $byId);
}
```

Build it once when its lifetime and freshness permit. If the catalog changes, define invalidation rather than accidentally using stale data.

## Security

Treat keys derived from users as untrusted data. Limit key length and collection cardinality. Do not interpolate keys into SQL, filesystem paths, shell commands, or log formats without the boundary-specific protection required there. A safe array key is not automatically a safe SQL identifier or path.

Canonicalization must be consistent across writing and reading. Case folding, Unicode normalization, whitespace, locale assumptions, and encoding can create duplicate-looking identifiers or authorization bypasses if different components normalize differently.

## Testing

Test collection invariants:

1. Verify insertion order, update behavior, and unset/reindex behavior where relied upon.
2. Test missing, null, false, zero, and empty-string values separately.
3. Test key canonicalization and numeric-looking identifiers.
4. Test duplicate-key policy: reject, overwrite, first-wins, or group.
5. Test large and adversarial key counts at the input boundary.

For an index, compare its results with a simple scan-based reference on generated small data. This catches incorrect normalization and silent overwrite behavior.

## Common Mistakes

- Calling every PHP array a list.
- Using `isset()` when null is a meaningful stored value.
- Assuming an associative lookup preserves duplicate records.
- Reindexing with `array_values()` without deciding whether keys matter.
- Using `array_shift()` as an unbounded queue implementation.
- Building a map with untrusted, unbounded keys.
- Assuming a short array syntax implies low memory overhead.

## Senior Engineer Thinking

An array is a language primitive; a map is a design intention. Make that intention visible in names, types, comments, and invariants. When the collection grows, ask whether PHP is the right owner of the index, whether a database index is more appropriate, and whether the data can be streamed instead.

The best map is not the one with the cleverest lookup. It is the one whose key identity, duplicate policy, lifetime, memory bound, and invalidation behavior are explicit.

## Exercises

1. Build a user index by ID that rejects duplicates and preserves a separate display order.
2. Create examples where `array_key_exists()`, `isset()`, and `??` return different results.
3. Group events by day and document ordering and memory behavior.
4. Replace an `in_array()` loop with an index. State when index construction is worthwhile.

## Review Questions

1. Why is a PHP array more than a vector?
2. When should `array_key_exists()` be preferred over `isset()`?
3. What happens to duplicate keys when building a map?
4. Why does key canonicalization belong to the algorithm’s contract?
5. Which array operations can create a second large array?

## Summary

PHP arrays are ordered maps that can serve many roles. Choose the role deliberately, distinguish presence from truthiness, define key normalization and duplicate policy, and account for key/value storage and temporary copies. Use a map for repeated lookup when its memory and freshness trade-offs fit the workload.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: `array_key_exists()`](https://www.php.net/manual/en/function.array-key-exists.php)
- [PHP Manual: `isset()`](https://www.php.net/manual/en/function.isset.php)
- [PHP Manual: `in_array()`](https://www.php.net/manual/en/function.in-array.php)
- [PHP Manual: `SplObjectStorage`](https://www.php.net/manual/en/class.splobjectstorage.php)
