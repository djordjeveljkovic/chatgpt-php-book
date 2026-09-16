---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 52
title: Copy-on-Write
slug: copy-on-write
status: complete
summary: ../../_ai/chapter-summaries/052-copy-on-write-summary.md
---

# Chapter 52 — Copy-on-Write

## Why This Matters

PHP code often appears to copy large arrays and strings at every assignment and function call. If that were literally true, ordinary application code would spend enormous amounts of time and memory copying data that is never changed. PHP instead uses copy-on-write (COW) for its main mutable value types: the assignment initially shares storage, and the engine separates the storage only when a write would make two values diverge.

COW is an optimization with observable performance consequences, not a change to PHP's value semantics. `$b = $a` still means that later writes to `$b` must not change `$a`; COW is how the engine postpones the work required to preserve that promise. Understanding the boundary between a cheap share and an eventual copy helps explain memory spikes, function design, array transformations, and long-running worker behavior.

## Mental Model

For an array or string, start with two zvals sharing one refcounted payload:

```text
$a = ["red", "blue"];
$b = $a;

zval($a) ──┐
           ├── refcounted array payload
zval($b) ──┘
```

On a write through `$b`, the engine separates the payload before changing it:

```text
zval($a) ─────── array payload ["red", "blue"]

zval($b) ─────── array payload ["red", "green"]
```

The language-level result is independent values. The implementation may use additional optimizations, but the invariant is the same: a write cannot unexpectedly change an independent by-value copy.

## What Is Shared?

In modern PHP, strings and arrays are the important userland COW types. Integers, floats, booleans, and null fit in a zval directly and do not need a heap payload to share. Objects use shared object identity rather than COW: assigning an object variable gives another variable access to the same object, and `clone` is the explicit operation for making a new instance. References introduce a different aliasing model and are covered in Chapter 51.

```php
<?php

declare(strict_types=1);

$left = 'a long message';
$right = $left;       // Share string storage initially.
$right[0] = 'A';      // Separate, then mutate.

$first = ['state' => 'new'];
$second = $first;     // Share array storage initially.
$second['state'] = 'ready';

var_dump($left, $right);       // a long message, A long message
var_dump($first, $second);     // new, ready
```

“Initially” is important. A profiler that inspects only the assignment line can miss the allocation that occurs later at the first write.

## What PHP Does at a Function Boundary

By-value parameters do not imply an eager deep copy of an array or string:

```php
<?php

/** @param array<string, int> $counts */
function total(array $counts): int
{
    return array_sum($counts);
}
```

Calling `total($largeCounts)` can share the array payload while the function only reads it. If the function mutates its local parameter, the engine must separate it when the caller still has another value sharing the payload. This is why passing a large array by reference merely to avoid copying is usually a category error: the default already defers the copy, while `&` changes mutation semantics and may create other costs.

The exception is an explicitly referenced variable. A by-reference parameter must bind to a variable and can mutate the caller's state; it is not a COW optimization switch.

## What Zend Does: Refcounts and Separation

At the implementation level, a zval contains a type and either inline data or a pointer to refcounted storage. Arrays and strings carry reference-counting metadata in their heap representation. A normal assignment copies the zval and increments the payload's ownership count. Before a write, engine code checks whether the payload is safely writable; if it is shared, it duplicates the relevant payload and writes to the duplicate.

The source-level shape is approximately:

```text
zval: type = IS_ARRAY, pointer = zend_array
zend_array: GC header, hash-table metadata, buckets, values
```

The exact macros, flags, and fast paths are internal. Current php-src exposes the COW-sensitive operations through helpers such as `ZVAL_COPY`, reference-count operations, and array separation routines in Zend headers. Extension authors must use the API for their target PHP version rather than hard-coding a structure layout.

The engine also has to account for references, typed properties, temporary values, immutable/interned strings, and garbage-collection candidates. “The refcount is two, therefore this line always copies” is too strong: the operation, value kind, and internal flags matter.

## Minimal Experiment

```php
<?php

declare(strict_types=1);

function change(array $input): array
{
    $input['changed'] = true;
    return $input;
}

$before = ['id' => 7];
$after = change($before);

var_dump($before); // ['id' => 7]
var_dump($after);  // ['id' => 7, 'changed' => true]
```

The function can use a by-value signature and still avoid copying until its write. `debug_zval_dump()` may help when exploring a particular build, but its output includes temporary references and is not a portable performance specification.

## Nested Values: COW Is Not Deep Cloning

COW protects the array container that is being written; it does not recursively clone every object or nested payload in advance.

```php
<?php

final class Label
{
    public function __construct(public string $value) {}
}

$one = [
    'tags' => ['php', 'runtime'],
    'label' => new Label('draft'),
];
$two = $one;

$two['tags'][] = 'memory'; // The nested array separates along the write path.
$two['label']->value = 'published'; // Both arrays reach the same object.

var_dump($one['tags']);          // php, runtime
var_dump($one['label']->value);  // published
```

The first mutation changes a nested array value without changing the original nested array. The second mutates one shared object. If a true independent object graph is required, implement a deliberate cloning/copying policy; COW does not provide it.

## Arrays, Strings, and Operations That Allocate

COW delays separation, but many operations intentionally create a result:

```php
$result = array_merge($first, $second);
$result = [...$first, ...$second];
$result = substr($text, 0, 100);
$result = str_replace('old', 'new', $text);
```

Whether an operation can reuse storage, returns an immutable/interned string, or allocates a new array depends on the operation and the version. Do not infer allocation behavior from the word “assignment” alone. In a hot path, measure a complete operation sequence, not just a refcount snapshot.

Repeated transformations can create temporary peaks:

```php
<?php

$rows = loadRows();
$rows = array_map(transformRow(...), $rows);
$rows = array_filter($rows, keepRow(...));
$rows = array_values($rows);
```

Each result may coexist with an input or temporary until the old value is released. A streaming generator or one-pass loop can reduce peak memory when latency and code clarity permit.

## Bad Example: Reference for “Speed”

```php
/** @param array<int, string> $items */
function upper(array &$items): void
{
    foreach ($items as &$item) {
        $item = strtoupper($item);
    }
    unset($item);
}
```

This may be correct if in-place mutation is the requirement. It is not automatically faster than a by-value function. The reference exposes the caller's array, makes aliasing harder to reason about, and can keep a large payload alive through a lingering alias.

## Better Example: Choose the Contract First

```php
<?php

/** @param list<string> $items @return list<string> */
function upperCopy(array $items): array
{
    foreach ($items as $index => $item) {
        $items[$index] = strtoupper($item);
    }

    return $items;
}
```

For a shared input, the first write triggers the COW separation and the caller keeps its original list. If the caller does not need the original afterward and peak memory is critical, an in-place API can be justified—but name and document it as such. A generator may be preferable when consumers can process one item at a time.

## Performance

Reason about COW with these costs:

```text
Assignment of a large array/string: often O(1) payload sharing
First write to a shared payload: can be O(n) for the separated structure
Read-only traversal: O(n) work, without requiring a payload copy
Result-building transformation: commonly O(n) time and O(n) additional space
```

These are models, not promises for every operation. Array buckets, hash-table capacity, nested values, allocator reuse, and cache locality affect real measurements. A single mutation near the end of a large array can still cause a full container separation. A tight loop that repeatedly creates intermediate arrays can have a much larger peak than its final result.

Benchmark with production-shaped data. Record wall time, peak `memory_get_peak_usage()`, request or worker RSS when relevant, and allocation behavior under the actual PHP build. Compare one-pass loops, COW-preserving reads, and streaming alternatives instead of optimizing by adding `&`.

## Long-Running Workers and Operational Failure

In a short web request, a COW allocation usually dies with the request. In a queue worker, shared values can live across jobs if stored in static state, a container singleton, or a retained closure. A job that first reads a large template and then mutates its local copy can produce a recurring memory peak. If references or object graphs retain both versions, reclamation can be delayed further.

Reset request-specific state at job boundaries, avoid caching mutable per-job arrays in global services, and measure memory over many iterations. Worker recycling is a useful containment strategy, but it does not replace finding an unintended retained value.

## Security

COW provides isolation between ordinary by-value arrays and strings, not a security boundary. Objects remain shared, references intentionally bypass separation, and serialization or external storage creates a different copy boundary. Do not rely on “the function receives an array by value” as a substitute for authorization or validation.

When handling secrets, release or overwrite application references where practical, avoid retaining large sensitive arrays in worker state, and remember that allocator behavior and copies made by extensions may leave data in memory longer than the source variable's lifetime.

## Testing

Test values, not the implementation's current allocation strategy:

```php
<?php

function addFlag(array $input): array
{
    $input['flag'] = true;
    return $input;
}

$source = ['id' => 1];
$result = addFlag($source);

assert($source === ['id' => 1]);
assert($result === ['id' => 1, 'flag' => true]);
```

For performance tests, use a separate benchmark harness and state the PHP version, build flags, input size, operation count, and whether OPcache/JIT is enabled. Assert a memory budget only when it is an explicit operational requirement, and allow a tolerance for allocator and platform differences.

## Common Mistakes

- Saying that PHP “copies arrays on assignment” without mentioning delayed separation.
- Saying that COW is deep cloning; nested objects remain shared.
- Passing every large array by reference even when the function only reads it.
- Measuring only the assignment and missing the first-write cost.
- Treating `debug_zval_dump()` refcounts as stable API behavior.
- Ignoring temporary arrays and peak memory in a transformation pipeline.

## Senior Engineer Thinking

Start from the required ownership semantics:

```text
Need an independent value?  → by-value result; COW may defer the cost.
Need shared object identity? → object assignment, with explicit mutation rules.
Need caller mutation?       → narrow by-reference API, documented and tested.
Need bounded memory?         → one-pass or streaming design, then benchmark.
```

COW is most useful when it lets a clean value-oriented API remain cheap for read-heavy paths. It is not a reason to hide mutation or to promise a constant-memory operation that eventually separates a large structure.

## Exercises

1. Benchmark assigning a 1,000,000-element array, reading it in a function, and mutating one element in a function. Explain the timing and peak-memory differences.
2. Build a nested array containing strings, arrays, and an object. Copy it and classify each mutation as independent, shared, or explicitly cloned.
3. Rewrite a three-stage `array_map`/`array_filter` pipeline as a one-pass loop and compare peak memory.
4. Design an API for a 500 MB import buffer: value return, in-place mutation, chunks, or a stream. State the failure and observability trade-offs.

## Review Questions

1. What language guarantee does COW preserve?
2. Which PHP values commonly use COW, and how do objects differ?
3. Why can a read-only by-value function avoid a large copy?
4. Why is COW not deep cloning?
5. When can a one-pass or streaming design be better than relying on COW?

## Summary

Copy-on-write lets PHP share array and string payloads across ordinary assignments and by-value calls until a write requires separation. It preserves value semantics while avoiding unnecessary eager copies. COW does not clone nested objects, does not apply to object identity, and does not turn references into a safe optimization. Analyze first-write cost, temporary results, peak memory, and worker lifetimes; then select value, mutation, or streaming APIs based on the actual contract.

## Official References

- [PHP RFC: Implicit move optimization](https://wiki.php.net/rfc/implicit_move_optimisation)
- [PHP Manual: Function arguments](https://www.php.net/manual/en/functions.arguments.php)
- [php-src: zval copy and assignment helpers](https://github.com/php/php-src/blob/master/Zend/zend_execute.h)
- [php-src: zval copy constructors and destruction](https://github.com/php/php-src/blob/master/Zend/zend_variables.h)
- [php-src: core type definitions](https://github.com/php/php-src/blob/master/Zend/zend_types.h)
