---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 51
title: References
slug: references
status: complete
summary: ../../_ai/chapter-summaries/051-references-summary.md
---

# Chapter 51 — References

## Why This Matters

PHP references are one of the language features most likely to be described with the wrong mental model. A reference is not simply a C pointer, and it is not the mechanism that makes ordinary array arguments cheap. It is a language-level alias: two variable names can write to the same variable content.

That distinction matters in code that mutates caller state, in `foreach` loops, in legacy APIs, and when a reference interacts with array copying. A mistaken reference can survive longer than the line that created it and quietly change a later assignment.

This chapter explains the observable semantics first, then gives a useful approximation of the Zend representation. The approximation is deliberately not an ABI promise; internal structures and optimizations can change between PHP versions.

## Mental Model

For ordinary assignment, imagine two variable slots pointing at a value payload:

```text
$a = ["open"];
$b = $a;

variable slot $a ──┐
                   ├── array payload ["open"]
variable slot $b ──┘
```

The payload may be shared until a write requires separation. This is copy-on-write, covered in Chapter 52.

With an explicit reference, the two slots instead address one shared variable container:

```text
$a = "open";
$b =& $a;

variable slot $a ──┐
                   ├── reference container ── value "open"
variable slot $b ──┘
```

Writing through either name changes the value observed through the other name. Assigning a new value through either name changes the shared container's value; assigning a new value to the slot itself is not the same operation as changing an object handle.

## Core Concept: Three Uses of `&`

PHP uses the ampersand in three related but distinct declarations or expressions.

### Assigning by reference

```php
<?php

declare(strict_types=1);

$status = 'pending';
$alias =& $status;

$alias = 'confirmed';

var_dump($status); // string(9) "confirmed"
```

`$alias =& $status` makes `$alias` another name for the same variable content. It does not make a snapshot, and it does not mean that later assignments to `$alias` should be copied back by a special synchronization step. There is one aliased variable.

### Passing by reference

```php
<?php

declare(strict_types=1);

function confirm(string &$status): void
{
    $status = 'confirmed';
}

$status = 'pending';
confirm($status);

echo $status; // confirmed
```

The parameter is bound to the caller's variable. A by-reference parameter therefore changes the function's contract: the caller must provide a writable variable, and the function is allowed to change it. Passing by reference is about observable mutation, not about asking PHP to avoid copying a large value. Arrays and strings already use copy-on-write for ordinary by-value passing.

### Returning by reference

A function can return a variable by reference, but both the declaration and the receiving expression must use `&`:

```php
<?php

declare(strict_types=1);

$settings = ['timezone' => 'UTC'];

function &setting(array &$settings, string $key): mixed
{
    return $settings[$key];
}

$timezone =& setting($settings, 'timezone');
$timezone = 'Europe/Belgrade';

echo $settings['timezone']; // Europe/Belgrade
```

This is powerful and difficult to audit. A caller can retain an alias to internal state after the function returns. Prefer an explicit setter, a returned value, or a small value object unless the alias itself is an intentional API feature.

## What PHP Does

The language reference describes a reference as an alias that allows different variables to write to the same value. It also distinguishes references from objects: an object variable contains an object identifier, and assigning that variable copies the identifier rather than creating a variable alias.

That gives three visibly different cases:

```php
<?php

declare(strict_types=1);

$a = 10;
$b = $a;       // independent variable values
$b = 20;

$c = 10;
$d =& $c;      // aliases
$d = 20;

final class Counter
{
    public int $value = 0;
}

$first = new Counter();
$second = $first; // same object, different variable slots
$second->value = 1;

var_dump($a, $c, $first->value); // 10, 20, 1
```

The last pair shares an object, but `$second = null` would only replace the value in `$second`; it would not replace `$first`. Use `===` when identity is the question, and `clone` when a distinct object is required. Object representation is treated in Chapter 53.

## What Zend Does: A Useful Approximation

In current php-src, a reference is represented by an internal reference object (`zend_reference`) containing reference-counting metadata and a `zval val` payload. A `zval` whose type is `IS_REFERENCE` points to that container. The exact macros and layout are implementation details, but this model explains why reference identity is different from merely sharing a refcounted string or array payload:

```text
zval (IS_REFERENCE) ──> zend_reference
                         ├── GC/refcount metadata
                         ├── val: zval("pending")
                         └── typed-property source metadata, when applicable
```

When code assigns by reference, the engine arranges for both variable slots to use the same reference container. When code reads or writes a reference, the engine follows the container to its `val`. Reference-counting keeps the container alive while aliases remain.

This is not permission to inspect a `zval` from an extension and assume that every PHP release uses the same offsets. Use the public extension API and macros appropriate to the target version.

## References and Copy-on-Write

The following looks like an ordinary array copy but contains a reference:

```php
<?php

$original = ['name' => 'Ada'];
$name =& $original['name'];
$copy = $original;

$copy['name'] = 'Grace';
echo $original['name']; // Grace
```

The referenced array element is already an alias-bearing variable. Copying the array preserves that relationship for the referenced element, so a later write can affect both arrays. This is an important reason not to create references merely to “optimize” a loop. `unset($name)` removes the local alias; it does not undo the historical fact that the element is a reference container.

The classic loop hazard is shorter:

```php
<?php

$rows = ['first', 'second', 'third'];

foreach ($rows as &$row) {
    // Intentionally mutate each element, or do nothing.
}

unset($row); // End the alias before reusing the variable name.

foreach ($rows as $row) {
    echo $row, "\n";
}
```

Without `unset($row)`, `$row` remains an alias to the last element. A later by-value `foreach` assigns through that alias and can overwrite the last element. The loop variable has scope beyond the loop, so the cleanup is part of the construct's correctness, not just style.

## Passing by Reference Is Not “Passing a Pointer”

The callee cannot rebind the caller's variable name by assigning a reference to the parameter. It can change the value to which the parameter is bound:

```php
<?php

$left = 'left';
$right = 'right';

function changeValue(string &$value, string &$other): void
{
    $value = 'changed';
    $value =& $other; // Rebinds the local parameter, not the caller's name.
    $value = 'other changed';
}

changeValue($left, $right);

var_dump($left, $right); // changed, other changed
```

The reference relationship between parameter and caller remains a language-level binding, while reassigning the local parameter changes what that parameter denotes. This differs from a C++ reference-to-reference intuition and is one more reason to keep reference APIs small.

## Production Example: A Deliberate In-Place Normalizer

There are legitimate uses, such as updating a large mutable work structure when the API explicitly promises in-place behavior:

```php
<?php

declare(strict_types=1);

/** @param array<string, int> $counts */
function removeZeroCounts(array &$counts): void
{
    foreach ($counts as $key => $count) {
        if ($count === 0) {
            unset($counts[$key]);
        }
    }
}

$counts = ['paid' => 4, 'failed' => 0];
removeZeroCounts($counts);
```

The contract is clear: mutation is the purpose, the input must be a variable, and tests can assert the postcondition. If callers need both versions, return a new array instead; making one function secretly mutate and another secretly copy is an operational debugging problem.

## Edge Cases and Common Mistakes

### `unset` breaks a variable binding, not necessarily every alias

```php
$value = 1;
$alias =& $value;
unset($value);
$alias = 2;
echo $alias; // 2
```

The name `$value` is removed. The reference container remains reachable through `$alias`.

### Do not use `&` to make read-only arguments fast

For strings and arrays, a by-value argument normally copies the zval and shares the payload until mutation. A by-reference argument can require a reference-capable variable and can inhibit useful engine optimizations. Benchmark the actual workload; the semantic cost is already enough reason not to add it casually.

### `debug_zval_dump()` is a diagnostic, not an application API

It can expose implementation-oriented reference-count information, but temporary values and the diagnostic call itself affect what is observed. Output can vary across builds and versions. Test behavior through public semantics, not an expected refcount.

### References are not object cloning

`$b =& $a` aliases the variable. `$b = $a` where `$a` contains an object copies the object handle. `clone $a` creates another object and invokes clone semantics. Keep these three operations distinct in reviews.

## Performance

Reference operations add bookkeeping and make alias analysis harder for both humans and the engine. They can avoid a deliberate copy when the required result is mutation of the caller's variable, but ordinary arrays and strings do not need them to avoid eager copying. The performance question is therefore:

```text
Do I need caller-visible mutation?
    ├─ no  → return a value / use by-value semantics
    └─ yes → use a narrow, documented by-reference contract
```

Measure memory with a representative process and input size. `memory_get_usage()` reports PHP allocator-visible memory, not the complete resident set, and a single reference experiment rarely predicts a long-running worker's behavior.

## Security and Operations

An alias can bypass assumptions at an API boundary. If a validator retains a reference to caller data, later code can mutate the validated value. If a request-scoped array is stored in a long-lived worker through a reference, it can also extend the lifetime of sensitive data.

Do not retain references to request data in static properties, service singletons, caches, or queue metadata. Prefer immutable snapshots for audit records and explicit copies at trust boundaries. When references are necessary, document ownership, mutation, and lifetime.

## Testing

Test the contract, including the negative space:

```php
<?php

declare(strict_types=1);

function increment(int &$value): void
{
    ++$value;
}

$value = 4;
increment($value);
assert($value === 5);

$items = [1, 2, 3];
foreach ($items as &$item) {
    ++$item;
}
unset($item);
assert($items === [2, 3, 4]);
```

Add a regression test for any `foreach` by-reference loop, including a later loop that reuses the variable name. If a reference crosses an object or array boundary, assert whether a copy, alias, or shared object is intended. Avoid asserting `debug_zval_dump()` output.

## Senior Engineer Thinking

When reviewing `&`, ask:

1. What exact caller-visible mutation is required?
2. Could a return value or a small result object express the same operation?
3. What is the alias's lifetime, especially after a loop or callback returns?
4. Does an array copy contain reference-bearing elements?
5. Is the behavior covered by a test that fails if the alias leaks?

References are not inherently wrong. They are a sharp feature whose benefit is a specific mutation contract, not a vague promise of speed.

## Exercises

1. Write `normalizeTags(array &$tags): void` that trims tags in place, removes empty tags, and leaves the keys predictable. Add a test for the loop-variable cleanup.
2. Rewrite the function as `normalizedTags(array $tags): array` and compare the clarity and memory behavior for a 100,000-element input.
3. Create an array containing one referenced element, copy it, and test which writes are shared. Explain the result using the reference-container model.
4. Design a safer replacement for a function that returns an internal array element by reference. State whether the replacement uses a setter, a value return, or a value object.

## Review Questions

1. How does a PHP reference differ from copying an object handle?
2. Why does ordinary by-value array passing not imply an eager full copy?
3. Why should a by-reference `foreach` variable usually be unset afterward?
4. What does returning by reference expose to the caller?
5. Why is `debug_zval_dump()` unsuitable as a correctness test?

## Summary

A PHP reference aliases variable content. It is used for assignment, by-reference parameters, and by-reference returns, each of which changes an API's mutation contract. Ordinary assignments of arrays and strings rely on copy-on-write; references use a shared reference container and can preserve aliases inside copied arrays. Object assignment is different again: it copies an object handle, while `clone` creates a new object. Use references narrowly, document their lifetime, clean up loop aliases, and test observable behavior rather than internal refcounts.

## Official References

- [PHP Manual: References Explained](https://www.php.net/manual/en/language.references.php)
- [PHP Manual: Objects and references](https://www.php.net/manual/en/language.oop5.references.php)
- [php-src: `zend_reference` definition](https://github.com/php/php-src/blob/master/Zend/zend_types.h)
- [php-src: reference assignment and variable handling](https://github.com/php/php-src/blob/master/Zend/zend_execute.h)
