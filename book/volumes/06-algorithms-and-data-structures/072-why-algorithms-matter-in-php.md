---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 72
title: Why Algorithms Matter in PHP
slug: why-algorithms-matter-in-php
status: complete
summary: ../../_ai/chapter-summaries/072-why-algorithms-matter-in-php-summary.md
---

# Chapter 72 — Why Algorithms Matter in PHP

## Why This Matters

An algorithm is the strategy a program uses to transform input into output. A data structure is the shape in which that input and intermediate state are held. Together they determine how a program behaves as data grows.

PHP makes it easy to write a correct first version. It also makes it easy to hide an expensive algorithm inside a short loop, a convenient array function, or an ORM call. A page that handles 50 records can appear healthy while the same code becomes unusable at 500,000 records. The syntax did not become wrong; the relationship between work and input changed.

Algorithmic thinking gives a PHP engineer a way to ask better questions:

- What is the input size, and which dimension matters: rows, bytes, keys, or graph edges?
- Which work is repeated unnecessarily?
- Does the data need ordering, uniqueness, random access, or only one pass?
- Is the cost in PHP, in the database, in serialization, or across a network boundary?
- What happens when the process receives malformed, adversarial, or unexpectedly large input?

The goal is not to replace practical engineering with mathematics. It is to make scaling behavior visible early enough to choose a design deliberately.

## Mental Model

```text
requirements and constraints
          │
          ▼
choose data representation
          │
          ▼
choose algorithm and boundaries
          │
          ▼
estimate time, space, and failure behavior
          │
          ▼
implement → measure → revise
```

An algorithm is a decision about work. A data structure is a decision about access. If a task repeatedly asks “have I seen this identifier?”, a set-like representation makes membership the primary operation. If a task needs the next item to process, a queue makes removal order explicit. If a task needs the smallest deadline first, a priority queue expresses that requirement more directly than repeatedly sorting an entire list.

The same PHP array can represent a list, map, stack, queue, or set-like collection. That flexibility is useful, but it does not mean those uses have identical semantics or costs. The representation should follow the operation the application performs most often.

## Core Concept

Consider validating that a batch contains no duplicate email addresses. A direct implementation compares each item with every later item:

```php
<?php

declare(strict_types=1);

function hasDuplicateEmails(array $emails): bool
{
    $count = count($emails);

    for ($i = 0; $i < $count; $i++) {
        for ($j = $i + 1; $j < $count; $j++) {
            if ($emails[$i] === $emails[$j]) {
                return true;
            }
        }
    }

    return false;
}
```

This is easy to understand. In the worst case it performs roughly `n * (n - 1) / 2` comparisons. A set-like lookup changes the strategy:

```php
<?php

declare(strict_types=1);

function hasDuplicateEmails(array $emails): bool
{
    $seen = [];

    foreach ($emails as $email) {
        if (isset($seen[$email])) {
            return true;
        }

        $seen[$email] = true;
    }

    return false;
}
```

The second version uses additional memory, but it performs one membership check per item. The improvement is not “using a shorter PHP function.” It is changing the algorithm from repeated pairwise comparison to incremental membership tracking.

That example also leaves an important question open: what is an email’s identity? If input can differ only by case or surrounding whitespace, normalize it before using it as a key. Algorithmic correctness depends on domain rules as well as loops.

## How It Works

Start with the contract, not the container:

```text
input: up to 2,000,000 imported records
operation: reject repeated external IDs
order: preserve original order for the error report
memory: bounded worker memory
failure: report the first duplicate with its line number
```

The contract suggests a set of seen IDs plus a counter or stream position. It also exposes a tension: a set gives fast membership but grows with the number of distinct IDs. If the input is too large for memory, the solution may need database uniqueness, external storage, partitioning, or a sorted/streaming design. “Use a hash map” is not the complete answer.

### Naive, improved, and production designs

The naive design is often valuable as a reference implementation. It gives a simple behavior oracle for small inputs. The improved in-memory design is appropriate when the distinct-key set fits comfortably within the memory budget. A production design may move the constraint to a durable boundary:

```text
small batch          → in-memory set
large import         → database unique constraint or partitioned validation
ordered input only   → streaming state with bounded retention
repeated queries     → indexed lookup or precomputed map
remote source        → account for network latency and retry behavior
```

The algorithm is therefore part of system design. A local O(n) pass can still be the wrong solution if it requires loading a 40 GB file into a PHP worker. A database operation can be efficient when it uses a selective index and disastrous when it scans and sorts millions of rows. Complexity belongs to the actual boundary where work occurs.

### Correctness before cleverness

Before optimizing, define invariants. For the duplicate detector:

```text
after processing item i:
    $seen contains exactly the normalized IDs from items 0 through i
    if a duplicate has been returned, its second occurrence has been processed
```

The invariant makes the code reviewable. It also reveals bugs such as inserting an unnormalized ID, checking after insertion, or clearing state between chunks when global uniqueness is required.

## What PHP Does

PHP supplies flexible collection primitives and functions, but a function name does not remove the need to understand its contract. `array_unique()` creates a new array and has comparison rules that may not match domain identity. `in_array()` scans values; whether that is acceptable depends on input size and strictness. An associative array can provide set-like behavior for integer and string keys, but PHP array key conversion rules matter.

The language also gives value semantics, references, objects, generators, and exceptions. These affect algorithm design:

- copying a large array may share storage initially but can incur a copy on mutation;
- an object in a collection represents identity, not merely a copied record;
- a generator can reduce peak materialized input but does not make downstream state free;
- throwing on the first invalid record is different from collecting every error;
- a function that accepts `array` has already required the caller to materialize the whole collection.

The implementation must respect the PHP semantics established in earlier volumes while choosing a suitable access pattern.

## What the Runtime Does

The Zend Engine executes the loops, calls, comparisons, and hash lookups through runtime values and internal handlers. PHP arrays are ordered maps rather than a narrow vector type; their convenience has memory and operation costs. Chapters 47–50 explain zvals, HashTables, and PHP arrays internally. Chapter 74 will focus on memory complexity and the difference between a bounded algorithm and a merely linear one that allocates too much.

This runtime knowledge is useful for hypotheses, not for pretending that a single constant is universal. PHP version, build, value types, OPcache, allocator state, and the SAPI can change measured results. Start with the abstract access pattern, then measure the target workload.

## Database and Network Boundaries

An application often performs only a small amount of PHP computation around a much larger database operation. Ask where the data should be filtered, grouped, joined, or ordered:

```text
database can use an index and return 20 rows
    → let the database perform selective filtering

database must return 20 million rows for PHP to discard
    → revisit the query, index, or ownership of the computation
```

The database has its own algorithms and cost model. A PHP loop over 20 rows is usually clearer than forcing a generic query builder to express every transformation. A PHP loop over 20 million rows may be a sign that the computation belongs closer to the data, or that the work needs a batch/streaming pipeline.

For remote services, count round trips as algorithmic work. A loop that performs one HTTP request per record is not merely O(n) in a local sense; it is n network waits with timeout, retry, and partial-failure behavior. Batching, bulk endpoints, or a durable queue may change the dominant cost.

## Practical Example: Normalize, Validate, and Preserve Evidence

This service uses a map for membership while preserving the line where a duplicate was found:

```php
<?php

declare(strict_types=1);

final readonly class DuplicateId
{
    public function __construct(
        public string $id,
        public int $firstLine,
        public int $duplicateLine,
    ) {
    }
}

/** @return DuplicateId|null */
function firstDuplicate(iterable $rows): ?DuplicateId
{
    /** @var array<string, int> $firstLineById */
    $firstLineById = [];
    $line = 0;

    foreach ($rows as $row) {
        $line++;
        $id = strtolower(trim($row['external_id']));

        if (array_key_exists($id, $firstLineById)) {
            return new DuplicateId($id, $firstLineById[$id], $line);
        }

        $firstLineById[$id] = $line;
    }

    return null;
}
```

The function accepts `iterable`, so the caller can provide an array or a generator. It stores one integer per distinct normalized ID, not every original row. It also makes the error useful: a duplicate is an input-quality problem with evidence, not just a boolean.

## Bad Example

```php
foreach ($orders as $order) {
    foreach ($customers as $customer) {
        if ($order['customer_id'] === $customer['id']) {
            // build result
        }
    }
}
```

This may be correct for a tiny in-memory example. At scale it repeatedly searches the same customer collection. If the relationship is a database join, use a query with appropriate indexes. If both collections are already in PHP and the customer ID is the lookup key, build a map once and perform direct lookups.

## Better Example

```php
$customerById = [];

foreach ($customers as $customer) {
    $customerById[$customer['id']] = $customer;
}

foreach ($orders as $order) {
    $customer = $customerById[$order['customer_id']] ?? null;
    if ($customer === null) {
        continue;
    }

    // Build the result for this order and customer.
}
```

The map changes repeated search into one indexing pass plus lookups. It is better only if the memory cost is acceptable and duplicate customer IDs have a defined policy. A robust implementation should reject or explicitly resolve duplicate keys instead of silently overwriting them.

## Performance

Describe performance along at least four dimensions:

```text
time: how work grows with input
space: additional live state
allocation: how much memory is created and copied
boundary cost: database, filesystem, process, or network work
```

Then measure representative inputs. Include small, typical, large, and hostile cases. Record PHP version, input shape, OPcache state, memory limit, and whether the benchmark includes I/O. A microbenchmark that times only a loop cannot justify a redesign of a request dominated by a database call.

## Security

Unbounded input turns an algorithm into a resource-exhaustion surface. A quadratic duplicate check can be abused with a large batch; a map can be abused with millions of distinct keys; a hash-based lookup can be stressed with pathological input or expensive normalization.

Set byte, item, and time limits at the boundary. Normalize with a documented rule. Avoid using user-controlled values as file paths, SQL fragments, shell arguments, or dynamic class names merely because they are convenient keys. Algorithm choice is part of denial-of-service defense.

## Testing

Test behavior and scaling assumptions separately:

1. Verify empty, one-item, duplicate, and normalized-equivalent inputs.
2. Verify the result when keys are missing or malformed.
3. Compare an optimized implementation with a simple reference implementation on generated small inputs.
4. Add a bounded-input test for the import or request boundary.
5. Benchmark a workload representative of production, but keep performance tests separate from deterministic unit tests.

## Common Mistakes

- Optimizing syntax instead of changing the access pattern.
- Calling every PHP array operation “constant time” without checking its contract.
- Ignoring memory because the time complexity is linear.
- Loading a whole stream into an array before deciding whether streaming was required.
- Replacing a database join with nested PHP loops without measuring either design.
- Using loose comparisons when identity requires exact normalized values.
- Keeping a clever optimization that makes the invariant impossible to explain.

## Senior Engineer Thinking

Senior algorithmic judgment is mostly constraint management. It asks what must be preserved, which operation dominates, where state can live, what can be streamed, and what failure means. It also knows when not to optimize: a clear O(n²) reference implementation may be ideal for 20 items, while a map or index is justified when the input or request rate makes repeated search material.

The useful deliverable is not “this code is O(n).” It is a design whose cost, memory bound, data ownership, and failure behavior are explicit enough to operate.

## Exercises

1. Implement duplicate detection with pairwise comparison, a set-like map, and a database uniqueness constraint. State where each design keeps state.
2. Given two lists of 10,000 product IDs, compare nested search, a map, and sorting followed by a merge. Include memory and ordering requirements.
3. Take a function accepting `array` and redesign it to accept `iterable`. Identify which guarantees change.
4. Find a loop in a PHP application that performs one remote call per item. Estimate the cost of batching and describe the failure semantics you would need.

## Review Questions

1. What is the difference between an algorithm and a data structure?
2. Why can a linear algorithm still be unsuitable for production?
3. When is building an index in PHP better than repeated scanning?
4. Why must identity and normalization be defined before using values as keys?
5. Which costs belong to PHP, and which belong to the database or network boundary?

## Summary

Algorithms make scaling behavior and resource use explicit. Choose the data structure from the dominant operation, define invariants and identity rules, account for memory and external boundaries, and measure representative workloads. PHP’s concise collection syntax is a tool, not a complexity guarantee.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: Array functions](https://www.php.net/manual/en/book.array.php)
- [PHP Manual: `array_key_exists()`](https://www.php.net/manual/en/function.array-key-exists.php)
- [PHP Manual: `memory_get_usage()`](https://www.php.net/manual/en/function.memory-get-usage.php)
