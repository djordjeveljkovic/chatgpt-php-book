---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 76
title: Sets
slug: sets
status: complete
summary: ../../_ai/chapter-summaries/076-sets-summary.md
---

# Chapter 76 — Sets

## Why This Matters

A set represents membership: an item is either present or absent. It does not primarily represent order, a value per key, or the number of times an item occurs. Making that distinction explicit simplifies duplicate detection, authorization checks, feature flags, graph traversal, joins, and cache invalidation.

PHP does not require a separate built-in `Set` syntax for common scalar-key cases. An associative array with a canonical key and a marker value is often enough. But a set has semantics that should be designed deliberately: what counts as the same item, whether order matters, whether duplicates are discarded, how objects are identified, and how much memory the membership index may consume.

## Mental Model

```text
set
├── membership: contains(x)
├── insertion: add(x)
├── removal: remove(x)
└── uniqueness: one logical x
```

For scalar identifiers:

```php
$permissions = [
    'invoice.read' => true,
    'invoice.write' => true,
];

$canWrite = isset($permissions['invoice.write']);
```

The values are markers. The keys are the set. If a null marker is possible, use `array_key_exists()` instead of `isset()`.

## Core Concept

Set membership changes repeated comparison into indexed access:

```php
/** @param iterable<string> $allowed */
function filterAllowed(iterable $requested, iterable $allowed): array
{
    $allowedSet = [];
    foreach ($allowed as $permission) {
        $allowedSet[$permission] = true;
    }

    $result = [];
    foreach ($requested as $permission) {
        if (isset($allowedSet[$permission])) {
            $result[] = $permission;
        }
    }

    return $result;
}
```

If `r` permissions are requested and `a` are allowed, building the set and filtering is approximately O(a + r) average time with O(a) additional space. Scanning the allowed list for every request is O(ar). The result preserves requested order and may contain duplicates. If the output itself must be unique, the result needs its own set or a different insertion policy.

## How It Works

### Define equality

A set is only as correct as its equality rule. These may represent the same domain permission:

```text
Invoice.Read
invoice.read
 invoice.read
```

If the domain says they are equivalent, normalize before insertion and lookup:

```php
function permissionKey(string $permission): string
{
    $key = strtolower(trim($permission));
    if ($key === '') {
        throw new InvalidArgumentException('Permission cannot be empty.');
    }

    return $key;
}
```

Use the same function at every boundary. Normalizing only when building the set creates false negatives during lookup. Avoid locale-dependent behavior unless the domain explicitly requires it; identifiers usually need a stable, documented canonical form.

### Set, multiset, and list

These structures answer different questions:

```text
list:     [a, a, b]       preserves order and duplicates
set:      {a, b}          preserves membership only
multiset: a → 2, b → 1   preserves counts, not necessarily order
```

Do not use a set when the number of occurrences affects billing, rate limits, inventory, or analytics. Use a frequency map instead:

```php
$counts = [];
foreach ($events as $event) {
    $type = $event['type'];
    $counts[$type] = ($counts[$type] ?? 0) + 1;
}
```

### Set operations

For canonical scalar keys, union, intersection, and difference can be expressed by building maps:

```php
function intersection(array $left, array $right): array
{
    $rightSet = [];
    foreach ($right as $value) {
        $rightSet[$value] = true;
    }

    $result = [];
    foreach ($left as $value) {
        if (isset($rightSet[$value])) {
            $result[$value] = true;
        }
    }

    return array_keys($result);
}
```

This returns unique values in the order first encountered in `$left`. PHP functions such as `array_intersect()` have their own comparison and key-preservation rules; choose them when their documented semantics match the domain, not simply because the name looks right.

### Objects and identity

Objects are not ordinary integer/string array keys. If the set is “these exact object instances,” `SplObjectStorage` provides an object-to-data map and can be used as an object set:

```php
$visited = new SplObjectStorage();

if (!isset($visited[$node])) {
    $visited[$node] = true;
    // visit this exact instance
}
```

This is useful for graph traversal and cycle detection. It is different from a set of domain IDs. Two separate `User` objects with the same ID are different object instances, but they may represent the same domain entity. Decide whether identity is object handle, database ID, immutable value, or another canonical key.

## Practical Example: Authorization Membership

An authorization check can prepare a set once for a request:

```php
final readonly class PermissionSet
{
    /** @param array<string, true> $members */
    private function __construct(private array $members)
    {
    }

    /** @param iterable<string> $permissions */
    public static function from(iterable $permissions): self
    {
        $members = [];
        foreach ($permissions as $permission) {
            $key = permissionKey($permission);
            $members[$key] = true;
        }

        return new self($members);
    }

    public function contains(string $permission): bool
    {
        return isset($this->members[permissionKey($permission)]);
    }
}
```

This makes normalization part of the value’s boundary. It also makes the set immutable from the caller’s perspective. The class does not solve authorization by itself: it must be built from trusted policy, use the correct subject and tenant, and be invalidated when permissions change.

## What PHP Does

Associative arrays preserve insertion order, so a set-like array may accidentally expose order. Do not make authorization or correctness depend on that order unless the contract says it matters. If deterministic output is required, define ordering explicitly rather than relying on incidental construction order.

`array_unique()` removes duplicate values and preserves keys according to its documented comparison behavior. It is a list transformation, not the same thing as an indexed set. It can be appropriate when the desired result is a de-duplicated list and input is already materialized. For repeated membership checks, building a set once is usually a clearer representation.

`SplObjectStorage` is designed for object-to-data association and can be used as an object set. The optional data attached to an object is not the same as the object’s identity. If object graphs may be cyclic, a visited set prevents infinite traversal.

## Performance and Memory

Sets trade memory for membership speed. Their cost includes each canonical key, marker, hash-table entry, and temporary normalization value. A set of 10 million strings is not “10 million booleans”; the strings and map metadata dominate.

If membership is needed once, sorting or a database query may be better. If the allowed collection is stable across many requests, a cache or database index might move the set out of each worker. If the input is huge, partition it or use external storage rather than raising the PHP memory limit.

The order of operations also matters:

```text
build set once → many lookups → good reuse
build set per item → repeated allocation and hashing
build set from unbounded input → memory risk
```

Measure both the build phase and lookup phase. A set does not make normalization, serialization, or remote policy retrieval free.

## Security

Membership checks are security-sensitive when the set represents authorization, CSRF tokens, replay IDs, or allowed destinations. Canonicalize consistently, scope the set to the correct tenant and subject, and define expiry or invalidation. A fast lookup with the wrong equality rule is a fast security bug.

Do not put secrets into logs merely because they are set keys. Protect token sets from unbounded growth and use constant-time comparison where the protocol requires it; ordinary hash membership and cryptographic secret comparison answer different questions.

For request filters, distinguish a set of allowed values from a set of blocked values and define the default when the set is unavailable. Fail-closed behavior is often appropriate for authorization, but can be inappropriate for a non-security feature flag; the policy belongs in the contract.

## Testing

Test set behavior explicitly:

1. Add the same value twice and verify the intended uniqueness result.
2. Test canonical-equivalent values and malformed values.
3. Test membership of absent, null-like, empty, and boundary values.
4. Test set operations with duplicates and different input orders.
5. For object sets, test two instances with equal fields but different identities.
6. Test authorization sets across tenants, role changes, expiry, and cache invalidation.

Property-based tests are useful: intersection should contain only values present in both inputs; adding an item twice should not change membership; difference should contain no value from the right set.

## Common Mistakes

- Using a list and repeated `in_array()` for high-volume membership checks.
- Treating a set as a multiset and losing counts.
- Forgetting to normalize before both insertion and lookup.
- Using object identity when the domain requires entity identity.
- Relying on insertion order accidentally.
- Building a fresh set inside a per-item loop.
- Treating `array_unique()` as a universal set implementation.

## Senior Engineer Thinking

A set is a statement about identity: “these values are the same for this decision.” Make that statement reviewable. Name the canonicalization function, the lifecycle of the set, the invalidation policy, the memory bound, and the behavior when membership data is unavailable.

In production, the hard part is often not the O(1)-like lookup. It is ensuring that the set is complete, fresh, correctly scoped, and built from the same identity rules as the writer.

## Exercises

1. Implement a scalar set with add, contains, and remove operations and document key normalization.
2. Implement intersection and difference while preserving the left input’s first-seen order.
3. Traverse a cyclic object graph using `SplObjectStorage` and a visited set.
4. Design permission-set invalidation after a role change in a multi-worker PHP deployment.

## Review Questions

1. What does a set preserve, and what does it intentionally discard?
2. Why must equality be defined before choosing a set representation?
3. When is a frequency map more appropriate than a set?
4. How does an object-identity set differ from an ID set?
5. Which operational concerns arise when a membership set is cached?

## Summary

A set models membership and uniqueness. In PHP, an associative array is often an effective scalar-key set, while `SplObjectStorage` fits object identity. Define equality and normalization, distinguish sets from lists and multisets, account for memory and lifecycle, and test freshness and security scope when membership controls behavior.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: `array_unique()`](https://www.php.net/manual/en/function.array-unique.php)
- [PHP Manual: `array_intersect()`](https://www.php.net/manual/en/function.array-intersect.php)
- [PHP Manual: `SplObjectStorage`](https://www.php.net/manual/en/class.splobjectstorage.php)
