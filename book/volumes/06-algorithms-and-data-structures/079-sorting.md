---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 79
title: Sorting
slug: sorting
status: complete
summary: ../../_ai/chapter-summaries/079-sorting-summary.md
---

# Chapter 79 — Sorting

## Why This Matters

Sorting turns an unordered or differently ordered collection into a sequence that follows a rule. That sounds mechanical, but the rule often carries product meaning: which task appears first, how equal-ranked results are presented, whether a report is reproducible, and whether page boundaries remain stable between requests.

PHP makes sorting easy to call and easy to get subtly wrong. A sort can discard keys, compare mixed types unexpectedly, rely on an incomplete comparator, consume enough memory to exceed a request limit, or produce output that changes when the input order changes. Before choosing `sort()` or `usort()`, define what “before” means, how ties behave, and where the data should be ordered.

## Mental Model

Sorting is a permutation under an ordering rule. It should rearrange elements without inventing or losing values:

```text
input records + ordering contract → same records in a defined sequence
```

An ordering contract specifies:

- the fields or properties that determine order;
- ascending or descending direction for each field;
- how values of different types are compared;
- what happens when the sort fields tie;
- whether original keys and relative input order matter.

The sort does not make records unique. Two records may have identical sort fields and remain distinct. Sorting also does not make an unstable source deterministic: if two rows compare equal, preserving their original order only helps when that original order itself is defined.

## Core Concept

Consider tasks ranked by descending priority, then by creation time, then by a unique ID:

```text
priority descending, created_at ascending, id ascending
```

The ID is a final tie-breaker. If it is unique and canonical, every pair of distinct tasks has a defined relative order. This is useful for stable pagination, repeatable reports, and testable output.

There are two separate properties to consider:

- **Stability:** elements that compare equal keep their prior relative order.
- **Determinism:** the same logical input produces the same order, regardless of incidental input ordering or execution timing.

Stable sorting does not create determinism when the original order is arbitrary. A unique final key can define determinism without relying on stability. PHP's built-in array sorting functions preserve the relative order of equal elements as of PHP 8.0; before PHP 8.0 that order was undefined. If the input order comes from a database query without a complete `ORDER BY`, stability cannot supply a reliable tie order.

## How It Works

### Choose a comparison policy

The built-in sorting flags encode common policies:

```php
$numbers = ['12', '3', '20'];
sort($numbers, SORT_NUMERIC); // numeric order: 3, 12, 20

$files = ['file10.txt', 'file2.txt', 'file1.txt'];
sort($files, SORT_NATURAL | SORT_FLAG_CASE);
// file1.txt, file2.txt, file10.txt
```

`SORT_REGULAR` is the default and uses PHP's normal comparison behavior. For mixed types, that can produce surprising results; use an explicit policy when values may be integers, numeric strings, ordinary strings, nulls, or objects. `SORT_NUMERIC` compares numerically. `SORT_STRING` compares as strings. `SORT_NATURAL` compares digit runs naturally, while `SORT_FLAG_CASE` can be combined with natural or string sorting. `SORT_LOCALE_STRING` depends on the current locale and therefore should not be an accidental choice for identifiers, protocols, or reproducible machine output.

These flags are comparisons, not domain parsers. Natural ordering is not semantic-version ordering, and numeric comparison is not validation. Validate and normalize values at the boundary when malformed or mixed inputs are possible.

### Pick the function by key behavior

PHP sorting functions mutate the array passed to them and return `true` (the return type is `true` as of PHP 8.2). Their key behavior is part of the contract:

| Need | Function | Resulting key behavior |
| --- | --- | --- |
| Sort values ascending or descending | `sort()` / `rsort()` | Assigns new numeric keys |
| Sort values and keep key/value association | `asort()` / `arsort()` | Preserves keys with their values |
| Sort keys | `ksort()` / `krsort()` | Preserves key/value association |
| Custom value comparison, list output | `usort()` | Reassigns numeric keys |
| Custom value comparison, keyed output | `uasort()` | Preserves key/value association |
| Custom key comparison | `uksort()` | Preserves key/value association |

This makes the distinction concrete:

```php
$byId = [42 => 'pear', 7 => 'apple'];

sort($byId);
// ['apple', 'pear'] with keys 0 and 1

$byId = [42 => 'pear', 7 => 'apple'];
asort($byId);
// [7 => 'apple', 42 => 'pear']
```

Use `usort()` when the output is conceptually a list and old keys are irrelevant. Use `uasort()` when keys identify entries and must stay attached. The manual's [sorting overview](https://www.php.net/manual/en/array.sorting.php) compares the built-in functions and their key-preservation behavior.

### Write a correct comparator

A user comparator receives two values and must return an integer less than zero, zero, or greater than zero according to whether the first value sorts before, ties with, or sorts after the second. It must describe one consistent ordering. PHP's [comparison-function contract](https://www.php.net/manual/en/function.usort.php) casts non-integer return values to integers, so returning a fractional difference such as `0.4` becomes zero and falsely reports a tie.

Use `<=>` for typed scalar values rather than subtraction. Subtraction can overflow or become a float for large integer differences, and values with unrelated types should not be silently mixed. Here is a list sort with a complete, explicit order:

```php
<?php
declare(strict_types=1);

/** @param list<array{id: string, priority: int, created_at: int}> $tasks */
function orderTasks(array $tasks): array
{
    usort(
        $tasks,
        static function (array $left, array $right): int {
            return ($right['priority'] <=> $left['priority'])
                ?: ($left['created_at'] <=> $right['created_at'])
                ?: strcmp($left['id'], $right['id']);
        },
    );

    return $tasks;
}
```

The comparator first orders larger priorities first, then earlier timestamps, then IDs in bytewise string order. The domain should ensure IDs are unique and normalized. `strcmp()` is a technical ordering for strings, not a language-aware display collation. For human names, locale and Unicode collation rules may be a product requirement; choose and configure that policy explicitly.

Comparator correctness is a data-integrity issue. At minimum, ensure that comparing a value with itself returns zero, reversing arguments reverses the sign, and transitivity holds: if `a` sorts before `b` and `b` before `c`, `a` must sort before `c`. The comparison must not depend on randomness, the current time, mutable global state, or side effects. A comparator that violates these properties can produce inconsistent results and make output difficult to reproduce.

### Multiple columns and ties

For rows already shaped as arrays, `array_multisort()` can sort parallel sort-key arrays and the rows together:

```php
$tasks = [
    ['id' => 'task-b', 'priority' => 2, 'created_at' => 100],
    ['id' => 'task-a', 'priority' => 3, 'created_at' => 120],
    ['id' => 'task-c', 'priority' => 3, 'created_at' => 110],
];

$priorities = array_column($tasks, 'priority');
$createdAt = array_column($tasks, 'created_at');
$ids = array_column($tasks, 'id');

array_multisort(
    $priorities, SORT_DESC, SORT_NUMERIC,
    $createdAt, SORT_ASC, SORT_NUMERIC,
    $ids, SORT_ASC, SORT_STRING,
    $tasks,
);
```

The arrays are compared lexicographically: later arrays break ties in earlier arrays. Include only intentional tie-breakers. If rows are passed as another sort array, equal values in all earlier columns can cause the rows themselves to be compared as an additional criterion. The official [`array_multisort()` documentation](https://www.php.net/manual/en/function.array-multisort.php) documents the argument order, sort flags, and key behavior. For a comparator with complex domain rules, `usort()` is often easier to review.

## What PHP Does

The PHP array API offers in-place sorts for values, keys, and custom comparisons. `sort()` and `usort()` replace existing keys with consecutive numeric keys; `asort()` and `uasort()` keep keys associated with their values. `array_multisort()` can sort several arrays at once or a multidimensional array by columns; string keys are maintained while numeric keys are reindexed. These details can change later lookups or JSON output shape, so test the actual key contract.

Sorting is stable in PHP 8 and later: equal elements retain their original relative order. Before PHP 8.0, their relative order was undefined. Do not depend on a particular tie arrangement in a legacy runtime unless your application adds an explicit tie-breaker.

The sorting manual notes that functions operate on the array variable itself, not by returning a separate sorted result. PHP's copy-on-write model can initially share an array value, but mutation may require the runtime to separate storage. See [Chapter 52 — Copy-on-Write](../04-php-under-the-hood/052-copy-on-write.md) and [Chapter 75 — Arrays and Hash Maps](075-arrays-and-hash-maps.md). Avoid assuming that an in-place API means no extra peak memory in every call path.

## Practical Example: Stable, Repeatable Task Order

If the source records must keep their external associative keys, `uasort()` uses the same comparator while retaining those keys:

```php
$tasksById = [
    'a17' => ['id' => 'a17', 'priority' => 2, 'created_at' => 100],
    'b04' => ['id' => 'b04', 'priority' => 3, 'created_at' => 100],
];

uasort(
    $tasksById,
    static fn (array $left, array $right): int =>
        ($right['priority'] <=> $left['priority'])
        ?: ($left['created_at'] <=> $right['created_at'])
        ?: strcmp($left['id'], $right['id']),
);
```

After sorting, `b04` still identifies its task and the values iterate in priority order. If IDs are the array keys and the row's `id` is guaranteed to match its key, comparing that ID makes ties explicit. `uasort()` is not a way to preserve source position as a meaningful business property; store that position explicitly if it participates in ordering.

### Decorate, sort, undecorate

Some sort keys are expensive to derive, such as parsing a date string, computing a normalized name, or applying a collation transform. A comparator may be called many times, so calculating that key on every comparison repeats work. Decorate each row with its precomputed key, sort the decorated entries, and then extract the rows:

```php
$decorated = [];
$position = 0;

foreach ($records as $record) {
    $decorated[] = [
        // This lowercase key is suitable for ASCII identifiers, not linguistic collation.
        'normalized_name' => strtolower($record['name']),
        'position' => $position++,
        'record' => $record,
    ];
}

usort($decorated, static fn (array $a, array $b): int =>
    strcmp($a['normalized_name'], $b['normalized_name'])
    ?: ($a['position'] <=> $b['position'])
);

$sorted = array_column($decorated, 'record');
```

This computes each normalized key once and gives ties a deterministic position relative to this input. It also allocates decorated data, so the time saved must justify the extra memory. The example's ASCII lowercase transform is not full Unicode or linguistic collation; use the appropriate locale-aware facility when user-visible language ordering matters.

## Sorting Versus Database Ordering

If records come from a database, decide where the ordering belongs. For a page of search results, put the order in SQL before `LIMIT` and `OFFSET` or keyset pagination. Fetching an arbitrary page and sorting only that page in PHP cannot produce the globally correct first page. A database may satisfy an `ORDER BY` from an index or perform a sort as part of its plan; inspect the plan when scale matters. See [Chapter 104 — SQL for PHP Developers](../08-databases/104-sql-for-php-developers.md), [Chapter 108 — Indexes](../08-databases/108-indexes.md), and [Chapter 110 — Query Plans](../08-databases/110-query-plans.md).

Always use a complete order for pagination. For example, `ORDER BY created_at, id` is more reliable than ordering by `created_at` alone when multiple rows share a timestamp. Even then, concurrent inserts or updates can move rows between pages; keyset pagination and a defined consistency boundary address that separate problem (see [Chapter 119 — Pagination](../08-databases/119-pagination.md)).

PHP sorting is appropriate when the collection is already in application memory, the order is application-specific, or a small bounded result must be rearranged after retrieval. It is usually a mistake to load a large table just to sort it in PHP: the application pays transfer and allocation costs, and the database cannot apply the order before filtering or limiting.

## Streaming, Complexity, and Memory

A full global sort of arbitrary input generally needs to retain or spool the values before it can emit the first correctly ordered item. If the input is already ordered by the same contract, it can be consumed in order. A database cursor can reduce PHP-side materialization, but it does not prove that the database itself avoided buffering or sorting. For datasets larger than process memory, use database ordering, external sorting, or a bounded selection strategy instead of collecting every row in a PHP array. [Chapter 74](074-memory-complexity.md) and [Chapter 90](090-streaming-algorithms.md) discuss memory complexity and streaming; [Chapters 83–84](083-heaps.md) cover heaps and priority queues for retaining only a top subset.

For planning, comparison sorting is commonly analyzed as O(n log n) comparisons, though the worst case depends on the algorithm. The PHP Manual describes the implementation of `sort()` as an implementation detail and does not make its algorithm or complexity an API guarantee. Auxiliary storage, callback count, and peak memory also depend on the implementation and call path. A custom comparison's cost multiplies the sorting work. If it parses a date or performs a database lookup on every call, a nominally efficient algorithm can still be slow. Extract keys once, avoid I/O in the comparator, and benchmark with realistic collection sizes and value shapes.

PHP arrays have significant per-entry overhead compared with compact native buffers. Sorting a copy while the original remains live can raise peak memory, particularly when rows contain large nested values. PHP's copy-on-write can defer some copying, not promise that a second full representation is free. Include input, output, temporary columns, decorated records, and retained references in the memory budget. [Chapter 74 — Memory Complexity](074-memory-complexity.md) explains peak live memory.

## Security and Determinism

Sorting untrusted data can be a resource-exhaustion boundary. Cap the number and size of records, bound expensive normalization, and do not make comparator callbacks perform I/O. A user-supplied sort field also needs validation. SQL parameters bind values; they do not safely stand in for column names or `ASC`/`DESC` syntax. Map an allowlisted request token such as `created` to a fixed SQL expression and fixed direction, rather than concatenating arbitrary input into `ORDER BY`. See [Chapter 144 — SQL Injection](../10-security/144-sql-injection.md).

Avoid locale-sensitive sorting for machine identifiers, signatures, or reproducible cache keys unless locale is explicit and fixed. A locale change can change ordering. For output that is hashed, compared across workers, or used for cursor boundaries, specify normalization, collation, direction, null placement, and tie-breaker as part of the contract.

Sorting does not make an unsafe value safe to display or query. Escaping, SQL parameterization, and identifier allowlisting still belong at their respective output boundaries.

## Testing

Test the ordering rule and invariants, not the current sort implementation:

1. Assert adjacent output pairs are ordered by the complete comparator.
2. Assert the output contains every input record exactly once.
3. Include equal primary keys and confirm all explicit tie-breakers.
4. Test duplicate sort keys, nulls, numeric strings, mixed-case text, and empty input according to the documented input contract.
5. Verify whether original keys are retained or reset.
6. Run deterministic-order tests with the same logical records in different input permutations; a unique tie-breaker should produce the same result.
7. Check comparator laws on generated values: self-comparison is zero, reversed arguments reverse the sign, and sampled triples are transitive.

For a small dataset, compare the production implementation against a simple reference ordering whose comparator is easy to inspect. For database queries, verify the actual SQL ordering and include rows with duplicate primary sort values. A test that only checks one happy-path array can miss key loss, unstable tie assumptions, and pagination errors.

## Common Mistakes

- Relying on `SORT_REGULAR` for values with mixed types.
- Returning booleans or fractional values from a comparator instead of a negative, zero, or positive integer.
- Comparing integers by subtraction and risking overflow or float conversion.
- Sorting with only a non-unique key and assuming ties are deterministic.
- Using `usort()` when original associative keys carry identity.
- Forgetting that `sort()` and `usort()` reindex keys.
- Passing data to PHP for sorting after the database has already limited the rows.
- Putting database, network, random, or clock-dependent work inside a comparator.
- Assuming stable sorting fixes an unspecified input order.
- Loading an unbounded result set just to sort it in a PHP request.

## Senior Engineer Thinking

Sorting is an ordering contract that crosses layers. The domain defines meaningful precedence; PHP implements an in-memory permutation; the database can order rows before transfer and pagination; APIs and caches may depend on repeatable order; and memory limits constrain how many records may be held at once.

When reviewing a sort, ask: are the compared values normalized and typed, is the comparator consistent and side-effect-free, are ties handled explicitly, do keys need to survive, who owns the data volume, and can an index or ordered stream do the work earlier? The answer determines the API and boundary more than the syntax does.

## Exercises

1. Sort a list of invoice rows by status rank, due date, and unique invoice ID. State how missing due dates are ordered.
2. Sort an associative map by its values while preserving its string keys. Then use `usort()` and explain the key difference.
3. Build an `array_multisort()` example for score descending and submission time ascending. Add a final tie-breaker so output is deterministic.
4. Write tests that check a comparator's sign symmetry and transitivity for generated scalar records.
5. A report loads 200,000 rows, applies a `LIMIT` in PHP after sorting, and uses only `created_at`. Explain the ordering, memory, and pagination problems and move the responsibility to the appropriate layer.

## Review Questions

1. What is the difference between a stable sort and deterministic ordering?
2. Which PHP sorting functions preserve key/value association, and which reindex values?
3. Why can a comparator that returns a boolean or fractional difference silently treat distinct values as equal?
4. When should ordering happen in SQL rather than in PHP?
5. Why does full sorting arbitrary input conflict with emitting results immediately as a stream?
6. Why should a sort used with pagination have a unique final tie-breaker?

## Summary

Sorting rearranges values according to an explicit comparison contract. Choose a PHP sorting function based on whether it sorts keys or values and whether keys must stay attached. Use a pure comparator that returns an integer sign and defines consistent tie behavior. Stable ordering helps preserve prior ties in PHP 8+, but deterministic output still needs a defined input order or a complete tie-breaker. Put large-scale ordering and pagination in the database when it owns the dataset, and account for sort work, temporary storage, and peak memory in PHP.
