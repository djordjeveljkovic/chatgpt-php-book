---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 47
title: zvals
slug: zvals
status: complete
summary: ../../_ai/chapter-summaries/047-zvals-summary.md
---

# Chapter 47 — zvals

## Why This Matters

PHP variables can change type, hold arrays and objects, participate in references, and be passed through userland and extension code. The Zend Engine needs one runtime representation that can describe all of those values. That representation is the zval (pronounced “zee-val”).

Understanding zvals makes several language behaviors less magical:

- integers and booleans can be stored directly in a value slot;
- strings, arrays, objects, and resources need pointers to separately managed payloads;
- assigning a value is not the same as immediately copying a large array;
- mutation can trigger copy-on-write separation;
- an explicit PHP reference is a distinct runtime condition from two ordinary variables containing equal values.

The zval is an engine implementation detail. PHP code should rely on the language’s value and reference semantics, not on the size of a C struct or on private tag constants remaining unchanged.

## Mental Model

```text
zval
├── type tag + flags
└── value storage
    ├── inline scalar: int, float, bool, null
    └── pointer: string, array, object, resource, reference
```

For a refcounted payload:

```text
variable zval ──► zend_string / zend_array / zend_object / ...
                  ├── refcount and GC flags
                  └── payload data
```

The diagram is conceptual. Current PHP 8 source stores a `zend_value` union and type information in a `zval`, but the exact field arrangement, flags, and macros are private and version-sensitive.

## Core Concept

A zval answers two questions:

1. What kind of value is this?
2. Where is the value’s data, and how is it owned?

The scalar cases fit in the value storage itself. A string zval points to a `zend_string`; an array zval points to a `zend_array`, which is built around a HashTable; an object zval points to a `zend_object`. Those payload types have their own headers and lifetime rules.

An illustrative model looks like this:

```c
/* Conceptual pseudocode, not a copyable PHP ABI definition. */
struct zval {
    union {
        long integer;
        double floating;
        refcounted_payload *pointer;
    } value;
    type_tag type;
    flags flags;
};
```

In current php-src, the public-looking C names are `zval`, `zend_value`, and macros such as `Z_TYPE_P()` and `Z_LVAL_P()`. These are for engine and extension code compiled against a compatible PHP API, not a promise that application code can inspect memory safely.

## How It Works

### Scalars and payload pointers

```php
<?php

$count = 3;
$enabled = true;
$ratio = 0.5;
$name = 'Ada';
$items = ['php', 'sql'];
$report = new stdClass();
```

Conceptually:

```text
$count   zval: integer 3
$enabled zval: boolean true
$ratio   zval: float 0.5
$name    zval: pointer → zend_string("Ada")
$items   zval: pointer → zend_array(...)
$report  zval: pointer → zend_object(stdClass)
```

The pointer does not mean PHP exposes a raw pointer or that the payload is always allocated independently in the same way. It means the value’s data is represented by a managed runtime object rather than by the scalar bits in the zval itself.

### Assignment and sharing

When a refcounted value is assigned, the engine can share the payload until a write requires separation:

```php
$first = ['status' => 'open'];
$second = $first;

$second['status'] = 'closed';

var_dump($first['status']);  // open
var_dump($second['status']); // closed
```

Conceptually:

```text
after assignment:
    $first  ─┐
             ├──► shared array payload (refcount 2)
    $second ─┘

before mutation of $second:
    separate payload if it is shared

after mutation:
    $first  ───► original array
    $second ───► private array with status = closed
```

This is copy-on-write, not a guarantee that every assignment is free. Incrementing ownership metadata, checking for separation, and eventually copying a large structure all have costs. Chapter 52 examines copy-on-write directly.

### Objects are different

Objects have identity semantics:

```php
$a = new stdClass();
$a->state = 'open';

$b = $a;
$b->state = 'closed';

var_dump($a->state); // closed
```

Both variables refer to the same object identity. Assigning an object does not create an independent object through ordinary copy-on-write. Use `clone` when a separate object is intended, and understand that cloning has its own shallow/deep-copy rules (see Chapter 38).

### Explicit references

An explicit reference changes the variable relationship:

```php
$left = 10;
$right =& $left;
$right = 20;

var_dump($left); // 20
```

Conceptually, both variable containers point to a reference payload containing the current zval:

```text
$left  ─┐
        ├──► zend_reference ───► zval(integer 20)
$right ─┘
```

This is not merely two zvals with the same integer. It is an aliasing relationship. Reference handling can inhibit straightforward separation and makes APIs harder to reason about, which is why output parameters and incidental `foreach (&$value)` usage deserve particular care. Chapter 51 covers references in detail.

### Undefined and temporary values

The engine also uses internal states such as `IS_UNDEF` and temporary values while evaluating expressions. These are not ordinary application-level types. A temporary may hold an intermediate result until an assignment, return, or function call consumes it; cleanup must happen if an exception interrupts evaluation.

This is one reason an opcode diagram is incomplete without value-lifetime reasoning: an addition, array lookup, or function call can create, reuse, separate, or release zvals and their payloads.

## What PHP Does

PHP specifies observable type behavior, assignment semantics, object identity, references, conversions, and errors. For example, `===` compares type and value, while object identity and equality follow PHP’s object comparison rules. The language does not expose a supported “zval address” API to userland.

`var_dump()` is useful diagnostic output, but it is not a memory-layout viewer:

```php
var_dump([
    'integer' => 3,
    'string' => 'Ada',
    'array' => ['x' => 1],
]);
```

It tells you PHP-level types and structure. It does not prove refcounts, sharing, bucket layout, or allocation count.

## What Zend Does

For a PHP 8.4 source reference, [Zend/zend_types.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h) defines the core zval/value types and type macros. The refcounted header and common ownership helpers are in [Zend/zend_types.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h), while strings, arrays, objects, and references add their specialized structures in other Zend headers.

The exact zval layout is architecture-, build-, and release-sensitive. Internal type tags, flag bits, alignment, and macro expansion must not be copied from one branch into another. Extension authors should use the supported extension API for the target PHP version and rebuild/test for every supported ABI.

## Minimal Example

This example demonstrates that equal values do not imply shared variables:

```php
<?php

$original = ['count' => 1];
$copy = $original;
$copy['count']++;

printf("original=%d copy=%d\n", $original['count'], $copy['count']);
```

Expected output:

```text
original=1 copy=2
```

The useful conclusion is semantic: ordinary assignment preserves value independence after mutation. The likely implementation strategy is shared payload plus separation, but code should not depend on whether a particular small value was copied eagerly or optimized.

## Practical Example: Measuring a Value-Shape Hypothesis

If an import process uses too much memory, inspect the value shape rather than guessing from source-line count:

```php
<?php

$rows = [];
for ($i = 0; $i < 100_000; ++$i) {
    $rows[] = ['id' => $i, 'state' => 'new'];
}

echo memory_get_peak_usage(true), PHP_EOL;
```

The memory includes zvals, array/hash metadata, buckets, string payloads, allocator behavior, and process overhead. Replacing an array of associative rows with a streaming iterator or a database cursor may matter more than reducing one scalar field. Measure with the same PHP build and allocator settings:

```bash
php -d memory_limit=512M tools/measure-rows.php
```

Do not infer exact per-zval bytes from one `memory_get_usage()` delta; allocator arenas and unrelated runtime state affect the result.

## Bad Example

```text
$b = $a means PHP copied every nested array and string immediately.
```

This leads to an incorrect memory model and can encourage unnecessary rewrites. It also ignores that nested payloads can have separate sharing relationships and that a later mutation may be the point at which cost appears.

## Better Example

Ask which operation forces ownership or identity changes:

```text
assign a large value
    → likely share refcounted payload while read-only
mutate one branch
    → separate the mutable value if needed
mutate an object property
    → update shared object identity
pass by reference
    → create/maintain an aliasing relationship
```

Then test the application’s actual memory peak and latency. If the data is read-only, sharing may be helpful; if many branches mutate large arrays, a different representation or streaming design may be needed.

## Edge Cases

- `null`, booleans, integers, and floats do not have the same payload/lifetime behavior as strings or arrays.
- A string can contain NUL bytes; its length is tracked separately from C-string termination (Chapter 48).
- Arrays and objects can contain cycles or references, affecting destruction and garbage collection.
- Resources are handles to extension-managed state, not ordinary PHP objects.
- A reference can make two variables observe one mutation even when the underlying value is scalar.
- Internal functions may validate/coerce arguments through C APIs in ways that differ from a userland function under strict typing; verify against the target PHP version.
- Debug builds, tracing, sanitizers, and profilers can alter allocation and timing behavior.

## Performance

Value cost has several components:

```text
cost ≈ zval slots + payload allocation + ownership checks
        + hash/object metadata + copying/separation + destruction
```

A scalar loop can be cheap while an array of tiny associative rows is expensive because every row carries structural metadata and multiple values. Conversely, a large shared read-only payload may avoid repeated copying. Complexity, memory locality, allocation count, and external I/O all matter.

Measure peak memory, not only final memory. In a PHP-FPM pool, a worker’s high-water behavior can reduce the number of requests the host can serve even after some values are released. Long-running workers need extra attention because request-scoped data that remains reachable can accumulate.

## Security

Memory representation is not a security boundary. Never treat a zval dump or pointer-like diagnostic as safe to expose; it can leak data, addresses, paths, or object state. Avoid unsafe deserialization and validate data before it becomes a large in-memory graph. Memory exhaustion is a denial-of-service risk, so bound input sizes, stream large payloads, and set operational limits appropriate to the workload.

## Testing

Test semantics first:

1. Assert that ordinary array assignment remains independent after mutation.
2. Assert that object assignment preserves identity and that `clone` changes it.
3. Test explicit reference APIs and cleanup of references in loops.
4. Run memory experiments in a separate process with fixed input and configuration.
5. If an extension depends on zval internals, compile and test against every supported PHP version and ABI.

For a behavior fixture:

```php
<?php

$a = ['state' => 'open'];
$b = $a;
$b['state'] = 'closed';

assert($a['state'] === 'open');
assert($b['state'] === 'closed');
```

Use engine-level assertions only in a version-pinned extension or php-src test; private tags and field offsets are not application contracts.

## Common Mistakes

- Calling a zval a “PHP variable” without distinguishing the variable container from its value.
- Assuming every assignment recursively copies data.
- Applying copy-on-write rules to objects as if they were arrays.
- Treating an explicit reference as an ordinary shared immutable value.
- Reporting exact memory cost without recording PHP version, architecture, allocator, and data shape.
- Using private zval layout in portable extension or application code.
- Forgetting temporary cleanup on exception paths.

## Senior Engineer Thinking

When a value-heavy operation is slow or memory-hungry, trace the value’s journey:

```text
source assignment
    → zval type and payload
    → sharing/reference state
    → mutation or function boundary
    → separation/allocation/destruction
    → worker memory and latency
```

That model leads to useful decisions: stream instead of materialize, use a database for filtering, avoid accidental aliases, choose a more suitable representation, or accept a copy because the data is small and the simpler code is safer.

## Exercises

1. Draw the conceptual zval/payload graph for an array containing a string, an integer, and a nested array.
2. Write a test showing the difference between `$b = $a` and `$b =& $a`.
3. Compare peak memory for 100,000 associative rows, a packed list of scalar IDs, and a generator that yields IDs.
4. Create an object assignment and a `clone` assignment. State which mutations each branch observes.
5. Read the zval definitions in a pinned php-src branch and list three details that are implementation-specific rather than PHP language guarantees.

## Review Questions

1. What two kinds of information does a zval carry?
2. Why can a refcounted payload be shared after assignment?
3. Why does ordinary object assignment differ from array copy-on-write?
4. What does an explicit reference change?
5. Why is `memory_get_usage()` insufficient evidence for an exact per-zval size?

## Summary

A zval is Zend Engine’s tagged runtime value container. Scalars can be represented directly; strings, arrays, objects, resources, and references point to managed payloads with their own lifetime rules. Refcounting and copy-on-write can share values until mutation, while objects preserve identity and explicit references create aliases. The layout and flags are private, version-sensitive implementation details; reason from PHP semantics and verify memory hypotheses with controlled measurements.

## References

- [php-src PHP-8.4: `Zend/zend_types.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h)
- [php-src PHP-8.4: `Zend/zend_string.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_string.h)
- [php-src PHP-8.4: `Zend/zend_hash.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_hash.h)
- [PHP Manual: variable basics](https://www.php.net/manual/en/language.variables.basics.php)
- [PHP Manual: references explained](https://www.php.net/manual/en/language.references.php)
- [PHP Manual: object cloning](https://www.php.net/manual/en/language.oop5.cloning.php)
- [PHP Manual: `memory_get_usage`](https://www.php.net/manual/en/function.memory-get-usage.php)
