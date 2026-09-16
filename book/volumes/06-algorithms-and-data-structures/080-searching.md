---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 80
title: Searching
slug: searching
status: complete
summary: ../../_ai/chapter-summaries/080-searching-summary.md
---

# Chapter 80 — Searching

## Why This Matters

Applications search constantly: a request checks whether a permission is present, an import finds a row by external ID, or a service looks for a reservation that conflicts with a requested time. The same word describes very different operations. Scanning a list, looking up a key in a PHP array, searching an already sorted sequence, and querying an indexed database all have different costs and equality rules.

The useful question is not simply “How do I find this value?” Ask what collection is being searched, what makes two values equal, how many searches will run, whether the collection changes, and which layer owns the data. Those constraints determine whether a linear scan is fine, an index is worthwhile, or PHP should not be doing the search at all.

## Mental Model

Searching is a contract between a query and a collection:

```text
Collection + equality rule + search strategy → result
```

The result might be a yes/no answer, a position, a key, the first matching record, or every matching record. Make that result explicit. “Find this customer” is ambiguous if duplicate customer rows are possible or if IDs can differ by type or case.

The simplest strategy visits entries one by one and stops when it finds a match. A PHP associative array can instead use a key as a prebuilt lookup index. A sorted collection allows the search range to shrink quickly, but only if its ordering matches the query. Chapters 73 and 75 established the relevant complexity and map concepts; Chapter 81 develops binary search in detail.

## Core Concept

For an unsorted collection of `n` values, a linear search takes O(n) time in the worst case and O(1) auxiliary space if it keeps no result collection. It can stop after the first match, so a match at the beginning is O(1). A miss requires examining every value. Average cost depends on where matches tend to occur; do not assume an average without a workload model.

If many queries search the same stable collection, building a map from the searched value to a result can pay off. Building it takes O(n) expected time and O(n) additional space; each subsequent key lookup is expected O(1) under the usual hash-map cost model. For `q` queries, compare approximately O(qn) work for repeated scans with O(n + q) work for building and using an index. The index can also change duplicate behavior, use extra memory, and become stale when the source changes.

If values are already sorted using the same comparison rule, a search can repeatedly narrow the possible range; Chapter 81 explains that algorithm. Sorting an unsorted collection solely to answer one query usually adds unnecessary work. Sorting may be worthwhile when the collection will serve many searches, needs ordered output anyway, or can be maintained in sorted form. Include sort construction, updates, memory, and query count in the comparison.

## How It Works

### Scan values when the collection is small or short-lived

`in_array()` asks whether a value appears among an array's values. Its third argument selects strict comparison:

```php
<?php

declare(strict_types=1);

$allowedRoles = ['editor', 'reviewer', 'publisher'];
$requestedRole = 'reviewer';

$isAllowed = in_array($requestedRole, $allowedRoles, true);
```

Without `true`, `in_array()` uses loose comparison. That can report a match between values of different types. For identifiers, permissions, enum-like strings, and other values with a defined type, strict comparison makes the rule clearer and avoids accidental matches caused by type juggling. String comparison remains case-sensitive; if your domain treats identifiers case-insensitively, normalize them at a deliberate boundary rather than hoping the search function will do it.

A loop gives more control when the match requires several fields or a domain predicate:

```php
/** @param list<array{id: string, status: string}> $reservations */
function firstConfirmedReservation(array $reservations, string $wantedId): ?array
{
    foreach ($reservations as $reservation) {
        if (
            $reservation['id'] === $wantedId
            && $reservation['status'] === 'confirmed'
        ) {
            return $reservation;
        }
    }

    return null;
}
```

The function returns the first row satisfying both conditions. It does not establish that IDs are unique or that the row is current. If duplicates violate a business invariant, detect or prevent them at the layer that owns that invariant instead of letting iteration order silently decide which record wins.

### Ask whether you need the value, its position, or its key

`array_search()` searches values and returns the first matching array key, or `false` when no value matches. The key may be integer `0`, which is false-like, so compare the result strictly:

```php
$values = ['north', 'south', 'west'];
$key = array_search('north', $values, true);

if ($key !== false) {
    // $key is 0; the value was found.
}
```

Use `in_array()` when only membership matters. Use `array_search()` when you need the first key associated with a matching value. Neither function searches recursively through nested arrays. For a custom record predicate or a nested structure, write the traversal that states the intended behavior.

### Search keys when the collection is already indexed

PHP arrays are ordered maps, as Chapter 75 explains. If the query is a key, use keyed access or a key-existence check rather than scanning every value:

```php
$userById = [
    'usr_104' => ['name' => 'Mira'],
    'usr_205' => ['name' => 'Jon'],
];

$user = $userById['usr_205'] ?? null;
```

This answers a different question from `in_array()`: it looks up the value stored under a key, rather than asking whether the key-like value occurs among the array's values. Make the map's key normalization explicit. PHP converts some array keys when arrays are built, so validate and canonicalize external identifiers consistently; distinct input representations should not accidentally become the same key. Reuse the canonicalization policy from Chapter 75.

When values can be `null`, null coalescing cannot distinguish a missing key from a present key whose value is `null`. Use `array_key_exists()` when presence is the question:

```php
$cache = ['last_checked' => null];

var_dump(isset($cache['last_checked']));
// bool(false): isset() requires a non-null value.

var_dump(array_key_exists('last_checked', $cache));
// bool(true): the key is present.
```

The PHP Manual specifies that `array_key_exists()` checks only the first array dimension. Do not use it as a recursive “path exists” test. Also avoid passing `null` as its key: PHP 8.5 deprecates null array offsets and null keys to this function; use the intended empty-string key explicitly if that is your data model.

### Build an index for repeated searches

Suppose an import contains many orders, and each order must be joined to an in-memory customer collection by a stable string ID. Repeatedly scanning all customers is easy to write but does unnecessary work:

```php
foreach ($orders as $order) {
    foreach ($customers as $customer) {
        if ($customer['id'] === $order['customer_id']) {
            // Process this match.
            break;
        }
    }
}
```

With `q` orders and `n` customers, this can perform O(qn) comparisons. Build a map once when the data shape and duplicate policy permit it:

```php
$customerById = [];

foreach ($customers as $customer) {
    $customerById[$customer['id']] = $customer;
}

foreach ($orders as $order) {
    $customer = $customerById[$order['customer_id']] ?? null;

    if ($customer === null) {
        // Apply the missing-customer policy.
        continue;
    }

    // Process the matched customer.
}
```

This map uses O(n) additional space and gives expected O(n + q) total work under the usual hash-map model. Assignment overwrites earlier values with the same key: this code implements “last customer with this ID wins.” If duplicate IDs are invalid, throw while building the index. If every matching row matters, store a list of customers per ID. An optimization is safe only when its duplicate and ordering behavior preserves the required result.

The `null` check above is unambiguous only because valid customer values are arrays, not `null`. If the map intentionally stores nullable values and the program needs to distinguish null from absence, check `array_key_exists()` first. Presence and value are separate facts.

## What PHP Does

PHP offers built-in value scans (`in_array()` and `array_search()`) and key-based array operations. The Manual defines their observable behavior, but it does not promise a formal Big O bound for every operation. Treat linear scans as O(n) in the standard algorithmic model. Treat hash-map lookups as expected O(1) when reasoning about a well-formed keyed array, not as a universal latency guarantee. PHP arrays carry more memory overhead than a compact vector; Chapters 49, 50, and 75 cover that representation.

Strict comparison answers whether PHP values are identical under PHP's comparison rules. For objects, strict identity means the same object instance, not “two objects represent the same customer.” If the domain's identity is a business key, compare that key or build a map by a canonical form of it. Do not use an object instance comparison when equal-looking values can be different instances.

Case folding, Unicode normalization, leading zeroes, whitespace, locale rules, and numeric-string handling are domain policies. Normalize once, validate the normalized form, and use that same rule when indexing and querying. A search whose build and query paths normalize differently may be fast and consistently wrong.

## Sorting and Search Trade-Offs

Sorting changes the cost profile, not the meaning of the data. If a collection is already sorted, Chapter 81 shows how binary search uses that order to find a value in O(log n) comparisons. If it is not sorted, include the O(n log n) comparison-sort cost described in Chapter 79 before claiming that repeated binary searches are cheaper. For `q` searches, the rough comparison is:

```text
repeated linear scan:             O(qn)
sort once, then binary search:     O(n log n + q log n)
build a hash map, then look up:    O(n + q) expected, O(n) extra space
```

These expressions omit constants and the cost of maintaining the collection. A map is attractive for exact key lookup, while sorting is useful when ordered traversal or range queries also matter. A sort can disturb input order, and maintaining a sorted structure as records change has a cost. If the collection is small or only searched once, a direct scan may be both faster in practice and easier to maintain.

Binary search depends on a stable total ordering and precise decisions about equal values, insertion points, and duplicate results. It is not a drop-in replacement for a search with arbitrary business predicates. Chapter 81 handles its invariants and implementation separately.

## Runtime and Database Boundaries

Search in PHP when the data is already present in memory, the collection is bounded, and the decision belongs to application logic. A one-pass scan can also be appropriate for an iterable or generator when retaining the entire result set would waste memory. Stop as soon as the answer is known; do not call `iterator_to_array()` just to use `in_array()` if streaming can meet the requirement.

For persistent data or a large candidate set, send a selective query to the database instead of loading every row and filtering it in PHP. The database may use an index, scan many rows, sort, or choose another plan; a `WHERE` clause does not make a search automatically constant-time. Check the query plan for important paths (Chapters 108–111), select only needed columns, and consider how many rows cross the network and are hydrated in PHP. Keep database comparisons aligned with PHP's comparison policy: collations, case sensitivity, and null handling may differ.

If the same data is independently copied into a PHP index, decide who owns updates and freshness. Rebuild or update the index when its source changes, and make cache invalidation explicit. A process-local map does not become shared or durable because it accelerates lookup; worker boundaries and persistence still apply (Chapters 64–65).

## Performance and Memory

For a linear scan, best-case time is O(1), worst-case time O(n), and auxiliary space O(1) when returning only a boolean or one match. Searching for every match necessarily uses space proportional to the result if all matches are retained. A keyed index adds O(n) space and has a build cost; when built once per request for one query, that cost may exceed the saved comparisons. If retained for many requests, include its lifetime, refresh cost, and memory in the decision.

The total request cost also includes input decoding, database or network latency, object hydration, comparator logic, and result construction. A loop over 100 values is rarely worth complicating for asymptotic reasons, but an O(nq) loop in a batch import can dominate quickly. Measure representative sizes and query counts. Track peak memory as well as elapsed time, because building a second map of a large result set can exhaust a worker before CPU becomes the problem.

## Production Concerns

Searching is often part of a correctness boundary. Authorization checks must use the current permission set and the intended identity rule; a stale or loosely compared list can grant access incorrectly. Import matching must define what happens when IDs are absent or duplicated. Reservation availability should use the database's consistency and concurrency controls when multiple requests can create reservations at once; a prior PHP-side search does not prevent a race.

Bound untrusted input before scanning or indexing it. A linear scan over attacker-controlled `n` values costs O(n), while an index may retain every submitted value and consume O(n) memory. Validate maximum lengths and counts, and avoid repeatedly rebuilding the same index inside a loop. Normalize and validate identifiers before using them as PHP keys. These rules reduce both accidental mismatches and resource exhaustion.

## Testing

Test the search contract, including:

1. Empty input, a match at the first and last positions, and a missing value.
2. Duplicate values and which match or key the API returns.
3. Strict versus loose comparison for values such as `7` and `'7'`.
4. A match at numeric key `0` from `array_search()`, checking with `!== false`.
5. A present key whose value is `null`, compared with an absent key.
6. Canonicalization cases such as case, whitespace, and numeric-string IDs.
7. Duplicate policy when building a map, and index refresh when source records change.

For an optimized search, generate small collections and compare its answers with a simple reference scan. This catches mismatched equality and duplicate behavior without relying on timing assertions. Benchmark separately with realistic collection sizes, hit/miss ratios, and query counts.

## Common Mistakes

- Using `in_array()` without strict mode when type differences matter.
- Searching values when the collection is already keyed for direct lookup, or checking keys when the requirement is value membership.
- Treating `isset()` as a pure key-presence check when `null` is a valid stored value.
- Treating `array_search()` result `0` as failure because it is false-like.
- Assuming a map is free: index construction, duplicate policy, stale data, and O(n) memory matter.
- Sorting before one search without including the sorting cost.
- Loading a large database table into PHP to filter it when a selective database query is more appropriate.
- Assuming a PHP-side pre-check protects a database invariant under concurrent requests.
- Applying one normalization rule while building an index and another while querying it.

## Senior Engineer Thinking

Start with the result the caller needs, then specify equality, duplicates, and freshness. Only after that compare the cost of scanning, indexing, sorting, and querying the database. A good review comment names both the workload and the semantic change: “This does 20,000 full scans of the same customer list. We can build an ID map once, but the source has duplicate IDs; should the import reject duplicates or preserve all matches?”

That question avoids optimizing into a different program. The best search strategy is the cheapest one that preserves the data model, ownership boundary, and correctness guarantees.

## Exercises

1. Write a function that returns the first index of a strict match in a list, or `null` if no match exists. Explain how your return type distinguishes “not found” from index `0`.
2. Given 50,000 customers and 30,000 orders, compare a nested scan with an ID index. Specify duplicate and missing-ID behavior before choosing the implementation.
3. Build an index that keeps every record for duplicate IDs. Return the matching records in original input order.
4. Describe a database query that should replace loading a full table into PHP. Identify the predicate, expected index, rows returned, and comparison/collation policy to inspect.
5. Implement a stream scan over an `iterable` that stops on the first match. Explain its time and memory bounds for a hit near the end and for a miss.

## Review Questions

1. What does a linear scan cost in the best and worst cases, and what changes the average case?
2. How does value membership differ from key lookup in a PHP array?
3. Why does a repeated-search workload sometimes justify an index, and what extra costs does it add?
4. Why should `array_search()` results be compared with `!== false`?
5. When do `isset()` and `array_key_exists()` answer different questions?
6. Why is sorting worthwhile only under certain search and update workloads?
7. What can make a PHP-side search incorrect even when its algorithm is fast?
8. When should a search move to the database, and what does an index fail to guarantee?

## Summary

A search needs a defined result and equality rule. Linear scans are simple, use constant auxiliary space, and take O(n) worst-case time. PHP array keys support expected O(1) average lookup when a suitable index already exists, at the cost of build time and O(n) memory. Sorting helps when ordering or many searches justify its cost; Chapter 81 covers binary search. For persistent, large collections, let the database search where that data is owned and inspect its plan. Test null presence, strict equality, duplicate policy, key normalization, and freshness alongside the algorithm.

## References

- [PHP Manual: `in_array()`](https://www.php.net/manual/en/function.in-array.php)
- [PHP Manual: `array_search()`](https://www.php.net/manual/en/function.array-search.php)
- [PHP Manual: `array_key_exists()`](https://www.php.net/manual/en/function.array-key-exists.php)
- [PHP Manual: `isset()`](https://www.php.net/manual/en/function.isset.php)
- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [Chapter 73 — Big O](./073-big-o.md)
- [Chapter 75 — Arrays and Hash Maps](./075-arrays-and-hash-maps.md)
- [Chapter 79 — Sorting](./079-sorting.md)
- [Chapter 81 — Binary Search](./081-binary-search.md)
