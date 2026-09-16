---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 17
title: Arrays
slug: arrays
status: complete
summary: ../../_ai/chapter-summaries/017-arrays-summary.md
---

# Chapter 17 — Arrays

## Why This Matters

PHP arrays are one of the language's most useful and most misleading features. The same syntax can represent a list, a dictionary, a record, a set, a queue, or a nested document. That convenience makes small scripts pleasant. It also makes it easy to choose the wrong representation, accidentally change keys, copy far more data than expected, or treat untrusted input as a trusted data structure.

An experienced PHP developer asks what the collection means before choosing an array operation:

- Is this an ordered sequence, a key/value map, or a record with a fixed shape?
- Are keys part of the data model, or are they incidental positions?
- Does order have business meaning?
- What must happen when a key is absent, present with null, or duplicated?
- How many elements are expected, and which operation is on the hot path?
- Should the database, a generator, or a dedicated object own this data instead?

An array is not a substitute for answering those questions.

## Mental Model

The most useful language-level model is that a PHP array is an ordered map: a collection of key/value pairs that can use integer and string keys. A list is a special use of that map in which keys are consecutive integers beginning at zero.

~~~php
<?php

declare(strict_types=1);

$list = ['red', 'green', 'blue'];
$map = ['status' => 'paid', 'attempts' => 2];
$record = ['id' => 42, 'email' => 'ada@example.test'];
~~~

These values are all arrays, but they have different contracts. A list is naturally iterated. A map is naturally looked up by key. A record has a known shape and may eventually deserve a value object or a dedicated DTO. Naming the shape in a docblock or type boundary is often more valuable than adding another helper function.

Order is observable. Insertion order affects iteration, JSON representations, and operations such as array_values(). Integer-looking string keys may be converted to integer keys under PHP's array-key rules, while other values cannot serve as ordinary array keys. Do not let incidental coercion define a public data format; normalize keys deliberately at a boundary.

## Core Concept

Array construction and access are simple, but the meaning of an operation depends on the shape:

~~~php
<?php

declare(strict_types=1);

$items = [
    ['sku' => 'book', 'quantity' => 2],
    ['sku' => 'pen', 'quantity' => 5],
];

$items[] = ['sku' => 'paper', 'quantity' => 1]; // append to a list
$items[0]['quantity'] += 1;                    // mutate nested data

$quantitiesBySku = [
    'book' => 3,
    'pen' => 5,
];

$bookQuantity = $quantitiesBySku['book'] ?? 0;
~~~

$items is a list of records. $quantitiesBySku is a map. Using $quantitiesBySku[] would express a different model and make lookup less clear.

For fixed-shape data, document the contract:

~~~php
/** @var list<array{sku: non-empty-string, quantity: positive-int}> $items */
~~~

Static analysis can then detect missing keys and invalid shapes before production. PHP's native array type alone cannot express these details.

## How It Works

### Reading, writing, and presence

These expressions answer different questions:

~~~php
$value = $data['name'] ?? 'anonymous';       // missing or null => fallback
$present = array_key_exists('name', $data);  // key exists, even if value is null
$notNull = isset($data['name']);             // key exists and value is not null
~~~

Use array_key_exists() when null is a meaningful value. Use isset() for the common “usable non-null value” case. A direct access can emit a warning for a missing key depending on the expression and PHP version, so validate external shapes before reading them.

### Iteration and mutation

foreach iterates values by default. The loop variable is a copy of the value at the language level, although copy-on-write can avoid copying storage until mutation. A reference loop variable has persistent consequences if it is not unset:

~~~php
foreach ($prices as &$price) {
    $price = round($price * 1.2, 2);
}
unset($price); // end the reference explicitly
~~~

The unset() matters. Without it, a later foreach ($prices as $price) can write through the lingering reference and corrupt the last element. Prefer returning a transformed array with array_map() or a normal indexed loop unless in-place mutation is actually required.

### Copy-on-write

PHP values use copy-on-write semantics for arrays. Assignment normally shares the underlying value until one variable changes it:

~~~php
$original = ['state' => 'draft'];
$copy = $original;

$copy['state'] = 'published';

// $original['state'] is still 'draft'.
~~~

This is useful, but it is not permission to pass huge arrays through many layers without considering memory. Mutation of a shared array can require a physical copy; nested arrays and repeated transformations can create substantial peak memory. References, debug_zval_dump(), and engine internals expose implementation details that should not become application contracts. The reliable contract is value-like behavior: changing one variable does not unexpectedly change another ordinary variable.

### Common operations

Choose operations by semantics, not by name similarity:

~~~php
$names = ['Ada', 'Grace', 'Linus'];

$upper = array_map(
    static fn (string $name): string => strtoupper($name),
    $names,
);

$long = array_filter(
    $names,
    static fn (string $name): bool => strlen($name) > 3,
);

$found = in_array('Ada', $names, true); // strict comparison is intentional
$position = array_search('Grace', $names, true);
~~~

array_filter() preserves keys, so $long may no longer be a list. Call array_values() when a dense list is part of the contract. array_search() can return integer zero, so compare its result with !== false, not with a truthiness test.

sort(), rsort(), and similar sorting functions reorder values and reindex numeric keys. asort() preserves keys while sorting by values; ksort() sorts by keys. array_unique() has comparison rules that may surprise readers, and it is not a general-purpose set abstraction. State the equality rule in code and tests.

## What PHP Does

PHP exposes arrays as values with built-in syntax and a large standard-library toolbox. It does not know whether an array is a DTO, a JSON document, a queue, or a set. It also does not enforce that all records in a collection have the same keys or types.

When a function receives an array, the function receives a value with PHP's copy-on-write behavior. A parameter declared as array accepts both lists and maps. Use a more precise boundary where the distinction matters:

~~~php
/** @param list<int> $numbers */
function total(array $numbers): int
{
    return array_sum($numbers);
}
~~~

If an operation requires a non-empty list, positive quantities, or unique identifiers, validate those invariants explicitly. A type declaration is a first line of defense, not a complete schema.

## What Zend Does

Internally, PHP arrays are implemented with Zend Engine hash-table machinery. A sequence with suitable integer keys can use a compact packed representation, while maps and arrays with irregular keys require more general hash-table storage. These are useful explanations for memory and performance behavior, but they are implementation details rather than application guarantees.

The practical consequences are stable enough to guide design: array lookup is generally efficient for a key, iteration is linear in the number of elements, and array storage has more overhead than a compact C-style element buffer. Copy-on-write delays copying but does not eliminate the cost of later mutation. Measure large workloads rather than relying on a slogan such as “arrays are O(1).”

## Minimal Example

This function normalizes an input list into a map keyed by SKU and rejects malformed records:

~~~php
<?php

declare(strict_types=1);

/** @param list<array{sku: string, quantity: int}> $lines */
/** @return array<string, int> */
function quantitiesBySku(array $lines): array
{
    $result = [];

    foreach ($lines as $index => $line) {
        if ($line['sku'] === '' || $line['quantity'] < 1) {
            throw new InvalidArgumentException("Invalid line at index {$index}");
        }

        $result[$line['sku']] = ($result[$line['sku']] ?? 0) + $line['quantity'];
    }

    return $result;
}
~~~

The input is a list because order and source position may matter for diagnostics. The output is a map because the next operation is lookup by SKU. Duplicate SKUs are combined deliberately rather than silently retained as duplicate list entries.

## Practical Example: Choosing a Shape

Imagine a reservation report with many rows. A first design might return this:

~~~php
[
    ['court' => 'A', 'date' => '2026-09-14', 'state' => 'confirmed'],
    ['court' => 'A', 'date' => '2026-09-15', 'state' => 'cancelled'],
]
~~~

That is suitable for ordered display. If the application repeatedly asks for the state of one court on one date, build an index explicitly:

~~~php
/** @param list<array{court: string, date: string, state: string}> $rows */
function indexReservations(array $rows): array
{
    $index = [];

    foreach ($rows as $row) {
        $key = $row['court'] . '|' . $row['date'];
        $index[$key] = $row['state'];
    }

    return $index;
}
~~~

Scanning the list for every lookup costs O(n) per lookup. Building the map costs O(n) expected time and O(n) additional space, after which key lookup is expected O(1). That trade-off is worthwhile when there are many lookups and the indexed key is unambiguous. If the source is a database table and the rows are large, an indexed SQL query may be better than loading all rows into PHP.

## Production Example: Streaming Instead of Accumulating

A report with millions of rows should not automatically become a millions-element array:

~~~php
/** @return Generator<int, array{email: string, total: int}> */
function reportRows(PDOStatement $statement): Generator
{
    while ($row = $statement->fetch(PDO::FETCH_ASSOC)) {
        yield [
            'email' => (string) $row['email'],
            'total' => (int) $row['total'],
        ];
    }
}
~~~

A generator lets the consumer process one row at a time, reducing application memory from “all rows plus array overhead” toward “one row plus buffers.” The database driver and query still have buffering behavior, so confirm the actual mode used by the driver. Streaming also changes failure behavior: an exception halfway through the iteration may leave a partial file or response. Write to a temporary destination and publish it only after successful completion when consumers require an all-or-nothing artifact.

## Bad Example

~~~php
<?php

function findUser(array $users, $id): array
{
    foreach ($users as $user) {
        if ($user['id'] == $id) {
            return $user;
        }
    }

    return [];
}
~~~

This mixes an unspecified input type, loose comparison, a linear scan on every lookup, and an ambiguous empty-array failure result. An ID such as a string with surprising numeric content can compare equal when it should not. The caller cannot distinguish “user exists with an empty-shaped record” from “not found.”

## Better Example

~~~php
<?php

declare(strict_types=1);

/** @param list<array{id: int, email: string}> $users */
function findUser(array $users, int $id): ?array
{
    foreach ($users as $user) {
        if ($user['id'] === $id) {
            return $user;
        }
    }

    return null;
}
~~~

If lookup dominates, change the input contract to array<int, array{...}> keyed by ID and use $users[$id] ?? null. If the data is authoritative and persistent, query the database by its indexed primary key. The better choice follows the workload, not a preference for one PHP idiom.

## Edge Cases

- isset($array['key']) is false when the key exists with value null; use array_key_exists() when that distinction matters.
- array_filter() keeps keys. Reindex a list explicitly with array_values().
- array_search() and strpos() can return zero; compare with !== false.
- foreach ($array as &$value) leaves a reference; unset($value) immediately afterward.
- array_merge() renumbers numeric keys, while the + union operator preserves left-hand keys and does not overwrite existing keys. They express different operations.
- JSON object/list interpretation depends on keys. A sparse numeric-key array may encode as an object rather than the list a client expects; normalize with array_values() when necessary.
- Recursive arrays cannot be safely serialized by every format, and recursive comparison or printing can fail. Validate external nesting and depth.
- Very large or attacker-controlled nesting can consume memory and CPU. Set input limits and validate size before recursively transforming data.

## Performance

For a collection of n elements, iteration and a single full transformation are O(n). Sorting is generally O(n log n), although the exact algorithm is an implementation detail. Key lookup in a hash map is expected O(1), not a mathematical guarantee for every pathological workload. in_array() and array_search() are O(n); strict comparison avoids coercion but does not change the asymptotic cost.

Space matters in PHP. A map may use substantially more memory than a packed list, and nested arrays multiply per-element overhead. Avoid repeated array_merge() inside a loop when a direct append or one final merge expresses the same result. For a hot path, profile peak memory and wall time with realistic data. If the required operation is aggregation, filtering, joining, or sorting of durable data, compare a database query with an in-memory implementation.

## Security

Arrays often carry attacker-controlled request data. Treat keys, nesting, and element counts as untrusted. Validate an allow-list of fields instead of copying an entire input array into a model or SQL parameter list. Never use array values to construct SQL, shell commands, filesystem paths, or HTML without the context-appropriate parameterization or escaping.

Be careful with authorization decisions based on membership. in_array($role, $roles) should normally use strict comparison, and the roles must come from a trusted source. A user-provided array called is_admin is still user input. When arrays contain secrets, avoid logging them wholesale; redaction must be structural and tested.

## Database Interaction

An array is an application representation, not a database index. Fetching a table and then filtering it in PHP can be correct for a small bounded set, but it can also turn an indexed query into a memory-heavy full scan. Push filtering, aggregation, and ordering to SQL when the database owns the data and can use indexes.

When passing a list to an IN predicate, generate one bound parameter per value and handle the empty-list case explicitly. Do not interpolate array values into SQL. When decoding JSON columns, validate the resulting shape before using it; a syntactically valid JSON object is not automatically a valid domain record.

## Concurrency

An array is process-local state. Two PHP requests do not share an array merely because they use the same source code, and mutating an array does not coordinate with another worker. If an array is used to enforce uniqueness, rate limits, inventory, or reservations, it is not a concurrency control mechanism. Use a database constraint, transaction, lock, or purpose-built shared store appropriate to the invariant.

## Testing

Test the shape and semantics, not only the final happy-path values:

~~~php
public function testDuplicateSkusAreCombined(): void
{
    self::assertSame(
        ['book' => 3],
        quantitiesBySku([
            ['sku' => 'book', 'quantity' => 1],
            ['sku' => 'book', 'quantity' => 2],
        ]),
    );
}

public function testMissingUserIsNull(): void
{
    self::assertNull(findUser([], 10));
}
~~~

Include cases for missing versus null, zero-valued search results, duplicate keys, sparse lists, invalid records, large inputs, and key order where order is part of the contract. Property-based tests are useful for invariants such as “all output quantities are positive” and “the total quantity is preserved.” Benchmark representative list and map sizes when memory or latency is a requirement.

## Common Mistakes

- Treating every array as a list and relying on numeric positions that later become sparse.
- Using loose comparison for identifiers or membership checks.
- Returning [] for every failure and losing the distinction between no result and an empty value.
- Calling array_filter() and forgetting that keys are preserved.
- Mutating by reference without unsetting the loop variable.
- Building a large in-memory collection when a cursor, generator, or SQL query would bound memory.
- Assuming copy-on-write means a large array is free to duplicate and mutate repeatedly.
- Trusting the shape of decoded JSON or request arrays without validation.

## Senior Engineer Thinking

Before writing an array expression, write down the collection's invariant: “ordered list of line items,” “map from SKU to positive quantity,” or “record with exactly these fields.” Then decide who owns filtering and lookup, what absent data means, and what size is realistic. A senior review asks whether the array is the right boundary at all. A typed value object, iterator, database query, or immutable result may communicate the contract more safely.

The best array code makes accidental states difficult to create. It names keys, normalizes lists, uses strict comparisons, and exposes failures. It also leaves a clear path to change the representation if a workload grows from hundreds of elements to millions.

## Exercises

1. Implement groupByStatus() for a list of records. Preserve input order within each group and document the resulting array<string, list<...>> shape.
2. Write two versions of a lookup: one scans a list and one uses an ID-indexed map. Measure their time and peak memory at 100, 10,000, and 1,000,000 records.
3. Build a function that accepts mixed decoded JSON and returns either a validated list of positive integers or a structured validation error. Include nested and sparse input cases.
4. Stream rows into a CSV file through a generator. Design the write path so a failure cannot publish a partial final file.

## Review Questions

1. Why is a PHP array better described as an ordered map than as a vector?
2. When do isset() and array_key_exists() produce different answers?
3. Why must the result of array_search() be compared with !== false?
4. What does copy-on-write optimize, and what cost can mutation still introduce?
5. Why can array_filter() unexpectedly change a JSON list into an object-like structure?
6. Which operations are O(n), which are expected O(1), and which are generally O(n log n)?
7. Why can a process-local array not prevent two requests from reserving the same resource?

## Summary

PHP arrays are ordered maps whose usefulness depends on the data shape they represent. Lists, maps, and fixed-shape records should be named and validated differently. Understand key presence, key preservation, strict comparison, copy-on-write, and reference behavior. For large data, choose an indexed query, generator, or streaming design when accumulation would exceed the memory or failure budget. Treat external arrays as untrusted input and make shared invariants the responsibility of a database or other concurrency-aware boundary.
