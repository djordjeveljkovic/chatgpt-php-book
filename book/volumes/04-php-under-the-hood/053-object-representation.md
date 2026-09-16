---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 53
title: Object Representation
slug: object-representation
status: complete
summary: ../../_ai/chapter-summaries/053-object-representation-summary.md
---

# Chapter 53 — Object Representation

## Why This Matters

An object variable does not contain an entire object inline in the variable slot. It contains a reference to an object managed by the engine. That fact explains why assigning an object does not copy its properties, why a method can mutate state visible through another variable, why `clone` is explicit, and why a shallow clone can still share nested objects.

It also explains several production concerns: object graphs can be large, circular graphs need garbage-collection support, dynamic properties have version-specific rules, and a long-lived worker can retain an object simply because one service or closure still holds a handle. The implementation model is valuable, but it must not be mistaken for a stable extension ABI.

## Mental Model

At userland level, separate variables can refer to the same object:

```text
$a = new Cart();
$b = $a;

variable $a ── object handle ──┐
variable $b ── object handle ──┴── Cart instance
```

The variables are not aliases. Replacing `$b` does not replace `$a`, but mutating the object reached through `$b` is visible through `$a`.

```php
<?php

declare(strict_types=1);

final class Cart
{
    public int $quantity = 0;
}

$a = new Cart();
$b = $a;
$b->quantity = 2;
$b = null;

var_dump($a->quantity); // 2
```

`$b = null` changes only the value held by `$b`; the object remains alive because `$a` still reaches it.

## Core Concept: Identity, State, and Class Metadata

A useful conceptual split is:

```text
Object variable (zval)
    └── object handle / pointer
          ├── instance state: declared and dynamic properties
          ├── class entry: class metadata and method table
          └── object handlers: operations supplied by the engine/class
```

Many instances of the same class share class metadata and method implementations. Their property values are per-instance. The method table is not copied into every object merely because each object can call the method.

The class entry contains metadata such as the class name, inheritance information, interfaces, properties, and functions. It is process/runtime state; the details and lifetime differ between normal request execution and persistent workers. The object contains instance-specific state and a link to its class entry.

## What PHP Does

Assignment copies access to the same instance. `===` tests whether two object variables identify the same object; it is not a deep comparison of all properties.

```php
<?php

final class Profile
{
    public function __construct(public string $name) {}
}

$original = new Profile('Ada');
$same = $original;
$different = clone $original;

var_dump($original === $same);      // true
var_dump($original === $different); // false

$different->name = 'Grace';
echo $original->name; // Ada
```

The `clone` operation creates a distinct instance and then invokes `__clone()` if the class defines it. Without a custom clone method, object properties are copied according to PHP's clone semantics; nested object properties still refer to the same nested objects.

## What Zend Does: `zend_object`

In current php-src, the core `zend_object` structure includes reference-counting metadata, a handle, extra flags, a pointer to `zend_class_entry`, object handlers, an optional properties hash table, and storage for declared properties. A simplified diagram is:

```text
zend_object
├── GC/refcount header
├── object handle
├── extra flags
├── ce ───────────────> zend_class_entry
├── handlers ────────> zend_object_handlers
├── properties ──────> dynamic/indirect property table, when needed
└── properties_table → declared-property storage
```

The real layout is version- and build-sensitive. The `properties_table[1]` notation in php-src is a C technique used with storage allocated after the structure; it is not a promise that every object has exactly one property slot. Property access also involves visibility, property metadata, typed-property state, hooks or handlers where supported, and possible magic methods.

The variable's zval carries the `IS_OBJECT` type and access to the object. Refcounting controls the instance lifetime. The class entry and handlers determine how operations such as property reads, writes, method calls, casts, comparisons, cloning, and destruction are performed.

## Properties: Declared Storage and Dynamic State

Declared properties are known to the class and can be associated with metadata such as visibility and declared type. The engine can store them in instance storage and distinguish uninitialized typed properties from initialized values.

Dynamic properties are a separate concern. In PHP 8.2 and later, creating undeclared dynamic properties is deprecated for most user-defined classes, with documented exceptions such as `stdClass`, `#[AllowDynamicProperties]`, and classes extending an allowed class. A modern class should normally declare its state rather than depend on accidental dynamic keys.

```php
<?php

declare(strict_types=1);

final class Invoice
{
    public function __construct(
        public readonly int $id,
        public int $totalCents,
    ) {}
}

$invoice = new Invoice(id: 10, totalCents: 2500);
// $invoice->totlaCents = 2500; // A typo should fail, not create state.
```

A `readonly` property prevents reassignment through the normal language rules after initialization; it does not make an object graph deeply immutable, and it does not make the whole object shared safely across threads or processes.

## Object Assignment Versus `clone`

```php
<?php

final class Line
{
    public function __construct(public string $sku) {}
}

final class Order
{
    /** @param list<Line> $lines */
    public function __construct(public array $lines) {}
}

$one = new Order([new Line('A-1')]);
$two = clone $one;
$two->lines[0]->sku = 'B-2';

echo $one->lines[0]->sku; // B-2: nested object is shared
```

If the business meaning requires an independent order, `__clone()` must clone the nested objects deliberately:

```php
<?php

final class DeepOrder
{
    /** @param list<Line> $lines */
    public function __construct(public array $lines) {}

    public function __clone(): void
    {
        foreach ($this->lines as $index => $line) {
            $this->lines[$index] = clone $line;
        }
    }
}
```

This is a domain decision, not an engine default. A deep copy can be expensive, can duplicate resources incorrectly, and can violate identity relationships elsewhere in the graph.

## Lifecycle and Destruction

When the last strong reference to an object disappears, the engine can release it; if a destructor exists, destruction semantics apply. Cycles complicate simple reference counting:

```php
<?php

final class Node
{
    public ?Node $next = null;
}

$a = new Node();
$b = new Node();
$a->next = $b;
$b->next = $a;

unset($a, $b);
```

The two nodes point at each other. A cycle collector can identify unreachable cyclic structures, but collection timing is not a substitute for closing files, releasing database transactions, or cancelling external work. `__destruct()` is not a reliable transaction boundary in a long-running service.

Object graphs can also contain closures, generators, resources wrapped by extensions, and framework containers. A retained root keeps everything reachable alive. Diagnose leaks by finding roots and ownership, not by assuming the object itself “forgot” to free memory.

## Production Example: Explicit Ownership

```php
<?php

declare(strict_types=1);

final class Report
{
    /** @param list<string> $rows */
    public function __construct(private array $rows) {}

    /** @return list<string> */
    public function rows(): array
    {
        return $this->rows;
    }
}
```

Returning the array by value gives callers a value-oriented snapshot with COW behavior until mutation. Returning internal references would expose object invariants. If rows contain mutable objects, decide whether the API exposes shared identity, immutable value objects, or a deliberate copy.

## Performance

Object assignment is usually cheap compared with copying a large object graph because it copies a handle and adjusts lifetime bookkeeping. That does not make object-heavy designs free. Each object has allocation, headers, properties, indirection, cache-locality, and garbage-collection costs. A million tiny objects can have a very different memory profile from one packed array of scalar records.

Cloning is at least proportional to the state copied and can be much more expensive for nested graphs. Method calls and property access can involve visibility/type checks, handlers, magic methods, and dynamic dispatch. Measure representative object counts and access patterns; do not optimize identity semantics away merely because an array benchmark is smaller.

## Security and Concurrency

An object handle is local to a PHP process. It is not a database identity, a stable serialization key, or a shareable pointer between FPM workers. Across a queue or HTTP boundary, serialize an explicit data contract and validate it; do not expect a cloned or serialized object to preserve live resources or authorization context.

Shared object identity inside one request can create time-of-check/time-of-use bugs when multiple services receive the same mutable object. Prefer immutable value objects or narrow commands at boundaries. In a long-running worker, reset mutable per-job objects and avoid storing request-specific object graphs in static roots.

## Testing

Test identity separately from state equality:

```php
<?php

final class Counter
{
    public int $value = 0;
}

$a = new Counter();
$b = $a;
$b->value++;
assert($a->value === 1);
assert($a === $b);

$c = clone $a;
$c->value++;
assert($c !== $a);
assert($a->value === 1);
```

Add tests for `__clone()` when nested objects or identity maps are involved. Test destructor-adjacent cleanup explicitly instead of relying on process shutdown. For worker code, run repeated jobs and measure retained roots or peak memory.

## Common Mistakes

- Saying object assignment copies the object.
- Using `clone` while assuming nested objects become independent.
- Treating `readonly` as deep immutability.
- Using object handles as cross-process identifiers.
- Relying on destructors for database commit/rollback or external cleanup.
- Assuming `zend_object` layout is stable across PHP versions.

## Senior Engineer Thinking

For every object boundary, name the ownership model:

```text
shared identity → mutation is visible to all holders
independent copy → clone/copy policy is explicit
immutable value  → replacement, not mutation, changes state
serialized DTO   → process boundary with validation
```

Then ask how identity affects caching, testing, concurrency, memory lifetime, and persistence. The engine makes handle sharing efficient; it does not decide whether sharing is correct for the domain.

## Exercises

1. Build a shallow and deep clone of an order with nested line objects. Write tests that show the difference.
2. Use `ReflectionClass` to inspect declared properties and compare that view with the conceptual `zend_object` diagram. Do not depend on private layout offsets.
3. Create a two-node cycle, remove the roots, and observe collection with `gc_status()`/`gc_collect_cycles()` in a controlled experiment.
4. Design a DTO for sending an object across a queue. List which fields are data, which are process-local resources, and which must be reloaded.

## Review Questions

1. What does an object variable contain conceptually?
2. Why does `$b = $a` share mutations but `$b = null` not replace `$a`?
3. What is the difference between declared property storage and dynamic properties?
4. Why is `clone` shallow for nested object references by default?
5. Why are object handles not valid cross-process identifiers?

## Summary

An object variable gives access to an engine-managed object instance. Assignment copies the handle, so mutations are shared while replacing one variable is independent. A `zend_object` links instance state to class metadata and object handlers; its exact layout is implementation-specific. `clone` creates a new top-level instance but does not automatically deep-clone nested objects. Design ownership explicitly, account for cycles and worker lifetimes, and test identity, state, cloning, and cleanup separately.

## Official References

- [PHP Manual: Objects and references](https://www.php.net/manual/en/language.oop5.references.php)
- [PHP Manual: The Basics](https://www.php.net/manual/en/language.oop5.basic.php)
- [PHP Manual: Cloning objects](https://www.php.net/manual/en/language.oop5.cloning.php)
- [PHP Manual: Deprecated dynamic properties](https://www.php.net/manual/en/language.oop5.basic.php#language.oop5.basic.class.dynamic)
- [php-src: `zend_object` definition](https://github.com/php/php-src/blob/master/Zend/zend_types.h)
- [php-src: standard object handlers](https://github.com/php/php-src/blob/master/Zend/zend_object_handlers.c)
