---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 73
title: Big O
slug: big-o
status: complete
summary: ../../_ai/chapter-summaries/073-big-o-summary.md
---

# Chapter 73 — Big O

## Why This Matters

Big O notation describes how the resource requirements of an algorithm grow as its input grows. It is a compact language for comparing designs before production traffic reveals the difference.

Big O is not a stopwatch. It does not tell you whether a function takes 2 microseconds or 2 seconds, and it does not include every database, network, allocator, or cache effect. Its value is that it highlights growth. A constant amount of work remains bounded as input grows; a quadratic amount of work can become the dominant cost surprisingly quickly.

## Mental Model

Let `n` represent the input dimension that drives the work. We usually care about an upper-bound growth class:

```text
O(1)       constant
O(log n)   repeatedly reduce the search space
O(n)       one pass over the input
O(n log n) divide-and-conquer or efficient comparison sorting
O(n²)      compare many pairs or nested full passes
O(2ⁿ)      enumerate subsets or branches
```

The notation ignores constant factors and lower-order terms for growth analysis. `3n + 40` and `n` are both O(n). That simplification is useful for large-scale comparison, but it is not permission to ignore constants when `n` is small or the operation is in a very hot loop.

## Core Concept

These functions find whether a value occurs in a list:

```php
function containsLinear(array $values, string $wanted): bool
{
    foreach ($values as $value) {
        if ($value === $wanted) {
            return true;
        }
    }

    return false;
}

function containsIndexed(array $valuesByKey, string $wanted): bool
{
    return array_key_exists($wanted, $valuesByKey);
}
```

The first function is O(n) per query. The second is approximately O(1) average-time lookup for a suitable map, after the map has been built. If there are `m` queries against the same values, scanning costs roughly O(mn), while building an index and looking up values costs roughly O(n + m) average time. The index also consumes O(n) additional space.

The better design depends on the workload. For one query over five values, building an index adds needless work. For 100,000 queries over the same 100,000 IDs, the index changes the problem materially.

## How It Works

### Identify the input dimensions

There may be more than one meaningful size:

```text
n = number of records
b = total bytes in text
u = number of unique keys
e = number of graph edges
q = number of queries
```

An algorithm can be O(n + b) or O(n + q), not merely “O(n).” A parser may do one operation per byte. A duplicate detector may use O(u) memory even when it reads n records. A graph traversal is often described in terms of vertices `v` and edges `e`.

Name the dimensions that affect the design. This prevents false confidence from choosing a convenient but incomplete `n`.

### Sequential and nested work

Two loops one after another usually add their costs:

```php
foreach ($orders as $order) {
    // O(n)
}

foreach ($refunds as $refund) {
    // O(m)
}
```

The total is O(n + m), often summarized as O(n) when both grow together. Nested loops usually multiply:

```php
foreach ($orders as $order) {
    foreach ($customers as $customer) {
        // O(n * m)
    }
}
```

But nesting alone is not enough to classify a loop. A two-pointer scan can have nested-looking control flow while each pointer only moves forward, giving O(n). Always count how many times each operation can execute, not just how many `foreach` statements appear.

### Best, average, and worst cases

Early returns produce different cases. Searching the first item is best-case O(1); finding a value at the end or not finding it is O(n). A hash-map lookup is commonly described as average O(1), but its worst-case behavior and actual constants depend on implementation, key distribution, resizing, and value handling.

State which case matters for the service. An authorization check that normally rejects at the first rule may have a different profile from a batch validator that must inspect every record. Worst-case limits matter when input is user-controlled.

### Amortized complexity

An individual operation can occasionally be expensive while a sequence has a good average cost. Dynamic storage may resize and move elements, but appending repeatedly can still have amortized O(1) behavior under the implementation’s growth strategy. The exact capacity policy is implementation-specific; the design lesson is to distinguish one operation’s worst case from a long sequence’s average total work.

Do not use amortized language to hide an unbounded requirement. A growing PHP array still needs memory for all retained values, and a reallocation can create a meaningful peak.

## What PHP Does

PHP exposes operations whose algorithmic costs vary by function and by data shape. A list scan such as `in_array()` is not equivalent to an associative-key check. Sorting is not a lookup index. `array_filter()` traverses input and returns another array, so the callback cost and result allocation both matter.

PHP arrays are ordered maps. They can act as lists or maps, but list-like syntax should not be treated as proof of a compact vector representation. Chapters 49 and 50 explain the internal HashTable and array behavior. For application reasoning, use the operation you need and verify the result with measurement when constants matter.

Strictness matters too:

```php
$values = ['10', 10, null];

$loose = in_array(10, $values);        // comparison policy is implicit
$strict = in_array(10, $values, true); // exact type/value matching
```

An algorithm can have the right growth class and still be incorrect because its equality rule is wrong.

## A Worked Comparison

Suppose `n` records each contain an ID and we need records whose IDs also occur in a second collection of size `m`.

### Nested scan

```php
foreach ($left as $leftRow) {
    foreach ($right as $rightRow) {
        if ($leftRow['id'] === $rightRow['id']) {
            // match
        }
    }
}
```

Time is O(nm), additional space is O(1) apart from the result. This may be appropriate when both collections are tiny and preserving duplicate match behavior is important.

### Index the right side

```php
$rightById = [];

foreach ($right as $rightRow) {
    $rightById[$rightRow['id']] = $rightRow;
}

foreach ($left as $leftRow) {
    $match = $rightById[$leftRow['id']] ?? null;
    if ($match !== null) {
        // match
    }
}
```

Average time is O(n + m), with O(m) additional space. But this changes duplicate semantics: if the right side has multiple rows for one ID, the last one overwrites earlier values. If all matches are needed, the value must be a list, and the total output size belongs in the cost model.

### Sort and merge

If mutation of order is acceptable, sort both collections by ID and advance the smaller current ID. Comparison sorting is typically O(n log n + m log m), while the merge pass is O(n + m). This can use less lookup metadata than a map in some environments, but sorting costs time and may require additional storage or copies. It is often useful for ordered streams or when inputs already arrive sorted.

There is no universally best algorithm. The constraints decide.

## Database Complexity

Do not write “the query is O(1)” because a `WHERE` clause exists. A database may use an index, scan a table, sort rows, join relations, or fetch many rows before filtering. Selectivity, cardinality, statistics, and the query plan matter.

The application-level model should include:

```text
PHP preparation + database execution + rows transferred + PHP post-processing
```

If a query returns `n` rows and PHP performs an O(n²) comparison over them, database optimization alone will not fix the application. If PHP sends n individual queries, the round-trip count may dominate even when each query is indexed.

## Performance

Use Big O to choose experiments, then measure with realistic data. Measure:

- wall-clock and CPU time;
- peak PHP-managed memory and, when relevant, process RSS;
- allocation and garbage-collection effects;
- number and latency of database or network calls;
- cold and warm behavior.

For a small input, an O(n²) algorithm with low constants can beat an O(n log n) design. For a large input, growth usually dominates. The crossover is an empirical property of the implementation and workload.

## Security

Complexity is a security property when an attacker controls input size or shape. Nested comparison, catastrophic regular-expression behavior, repeated remote calls, and unbounded maps can all create denial-of-service conditions. Set limits and test worst-case input, not only valid average traffic.

Be careful with hash-based structures and normalization. Keys that are huge strings consume memory; converting arbitrary input into keys can itself be expensive. Reject inputs outside the contract before building large intermediate state.

## Testing

Use tests to lock down semantics, not a guessed constant:

1. Test empty, one-item, duplicate, and high-cardinality cases.
2. Test equality and key-normalization rules explicitly.
3. Test duplicate-key policy when building indexes.
4. Use property-based or generated tests to compare an optimized algorithm with a simple reference on small inputs.
5. Keep benchmark thresholds environment-aware; a unit test should not fail because a shared runner is slower.

## Common Mistakes

- Saying “O(1)” without naming the average/worst-case assumptions.
- Counting source lines instead of executed operations.
- Forgetting index-construction cost.
- Ignoring output size, which can itself be Ω(k) for k returned items.
- Treating database query complexity as if it were PHP loop complexity.
- Optimizing a cold path while ignoring network or disk latency.
- Applying Big O to a single fixed-size configuration where measurement is more useful.

## Senior Engineer Thinking

Big O is a conversation tool. A good review comment connects the growth class to a product constraint: “This scans all permissions for every item; at our batch size and request rate that becomes millions of comparisons. Can we index permissions by code?” It also acknowledges trade-offs: the index consumes memory, changes duplicate behavior, and needs invalidation if the source changes.

The notation is most valuable when it leads to a clearer contract, a bounded design, and an experiment that can confirm or reject the hypothesis.

## Exercises

1. Classify the time and additional-space complexity of a duplicate detector with a map and with nested loops.
2. Analyze an algorithm with two inputs of sizes `n` and `m`; do not collapse them into one variable until you state the assumption.
3. Design two solutions for intersecting two ID lists: a map-based solution and a sort-and-merge solution. Compare ordering and duplicate behavior.
4. Inspect a production query and list the work that occurs in the database, over the network, and in PHP.

## Review Questions

1. What does Big O intentionally discard?
2. Why can sequential loops add while nested loops multiply?
3. What is amortized complexity, and what does it not say about memory limits?
4. Why must input dimensions be named explicitly?
5. Why is an average O(1) map lookup not a guarantee about total request latency?

## Summary

Big O describes growth, not exact elapsed time. Name the input dimensions, count executed work, include index construction and output size, distinguish average from worst case, and account for PHP, database, and network boundaries. Use the notation to choose measurements and expose trade-offs.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: `in_array()`](https://www.php.net/manual/en/function.in-array.php)
- [PHP Manual: `array_key_exists()`](https://www.php.net/manual/en/function.array-key-exists.php)
- [PHP Manual: Sorting arrays](https://www.php.net/manual/en/array.sorting.php)
