---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 38
title: Cloning
slug: cloning
status: complete
summary: ../../_ai/chapter-summaries/038-cloning-summary.md
---

# Chapter 38 — Cloning

## Why This Matters

An object variable is a handle to an object, not a value copy. Assignment creates another handle to the same object. `clone` asks PHP for a distinct object, but the default operation is shallow: scalar properties are copied and object-valued properties still point to the same nested objects.

That distinction matters for builders, snapshots, graph nodes, test fixtures, and mutable aggregates. A clone that shares a mutable collaborator can produce surprising cross-talk. A clone that duplicates a database connection or external resource can be worse.

## Mental Model

```text
$alias = $original       same object identity
$copy = clone $original  new outer identity, shallow property copy
                         └── nested objects shared unless __clone repairs them
```

Cloning is an ownership decision. Before writing `clone`, say what the new object owns, what it shares, and which identity fields must change.

## Minimal Example

```php
<?php

declare(strict_types=1);

final class Draft
{
    public function __construct(public string $title) {}
}

$one = new Draft('Morning court');
$alias = $one;
$copy = clone $one;

$alias->title = 'Changed through alias';
assert($one->title === 'Changed through alias');

$copy->title = 'Independent draft';
assert($one->title !== $copy->title);
assert($one !== $copy);
```

An object can be cloned only if its class is accessible and the operation is permitted. `__clone()` cannot be called directly like an ordinary public method; the `clone` expression triggers it on the new object.

## How It Works

PHP creates a new outer object and performs a shallow property copy. References to other variables remain references. If the class declares `__clone(): void`, PHP invokes it after the copy so the class can repair nested object state, reset identity, or establish clone-specific invariants.

```php
final class ReservationDraft
{
    public function __construct(
        public ?string $id,
        public array $notes,
        public DateTimeImmutable $startsAt,
    ) {
    }

    public function __clone(): void
    {
        $this->id = null;
        $this->notes = array_values($this->notes);
    }
}
```

`DateTimeImmutable` is safe to share for the usual value-object purpose. If the nested value were a mutable `DateTime` or a mutable collaborator, the clone policy would need to be different.

## Deep Copy Is Not a Universal Goal

There is no general “deep clone this graph correctly” rule. Some objects represent identity and should remain shared; some represent owned mutable state and should be copied; resources, sockets, locks, and service objects should usually not be copied at all.

```php
final class LineItems
{
    /** @param list<LineItem> $items */
    public function __construct(public array $items) {}

    public function __clone(): void
    {
        foreach ($this->items as $index => $item) {
            $this->items[$index] = clone $item;
        }
    }
}
```

This makes the collection own its line items, but it may be wrong if a `LineItem` refers to a shared product catalog or an entity identity. Define ownership in the domain instead of recursively cloning every object.

## Practical Example: Copying a Reservation Command

For an immutable readonly command, cloning normally adds little value because replacement is clearer:

```php
final readonly class ReservationRequest
{
    public function __construct(
        public string $courtId,
        public DateTimeImmutable $startsAt,
        public DateTimeImmutable $endsAt,
    ) {}
}
```

Use a named constructor or a `with...()` method when a modified value is needed. For mutable drafts, `clone` can express a branch:

```php
$candidate = clone $draft;
$candidate->notes[] = 'Requires lights';
```

The object must document whether the branch owns its notes and nested items. PHP 8.3 permits readonly properties to be reinitialized during `__clone()` on the clone, which supports carefully designed copy-with-changes objects.

## Bad Example and Better Example

Bad:

```php
final class ServiceState
{
    public function __construct(public object $cache) {}
}

$copy = clone $state;
$copy->cache->clear(); // also changes the original's cache
```

Better options are to inject a shared cache intentionally, clone an owned cache with an explicit `__clone()`, or make the state a value object that contains only data. Do not rely on the default shallow behavior accidentally.

## Identity, Persistence, and Concurrency

Cloning an entity must answer whether the new object is a new database row or a local alternate view. Reset identifiers for a new record only if the persistence layer expects insertion; otherwise a cloned entity may overwrite the original. Never infer database uniqueness from object identity. Two PHP processes can clone equivalent objects and persist conflicting rows.

Cloning does not provide a transaction, lock, or snapshot isolation. For a consistent database snapshot, ask the database for one under an appropriate transaction. A PHP clone only captures the in-memory values present at one point in one process.

## Performance

The outer copy is proportional to the number of properties copied; a custom deep copy may traverse the entire reachable graph, so time and memory are O(V + E) for the owned subgraph in graph terms. Cloning large arrays and object graphs can create significant allocation and garbage-collection pressure. A purpose-built projection or immutable value may be cheaper than cloning a service aggregate.

## Security and Operations

Do not clone objects holding credentials, open handles, locks, or request-scoped capabilities unless the class explicitly defines safe ownership. A clone can retain sensitive data longer than intended. In workers, repeated cloning of a large fixture can raise the process high-water mark even after temporary objects are released. Measure worker memory and use resettable factories or IDs where appropriate.

## Testing

Test identity and ownership separately:

```php
$original = new ReservationDraft(
    'r-1',
    ['created'],
    new DateTimeImmutable('2026-09-14 10:00 UTC'),
);
$copy = clone $original;

assert($copy !== $original);
assert($copy->id === null);
$copy->notes[] = 'changed';
assert($original->notes === ['created']);
```

For nested objects, mutate the clone's child and assert whether the original changes. Add tests for reference-valued properties, identity reset, readonly clone behavior, and failures in `__clone()`. These tests are more valuable than a blanket assertion that `clone` “works.”

## Common Mistakes

- Confusing assignment with cloning.
- Assuming `clone` performs a deep copy.
- Copying services, connections, resources, or locks without an ownership policy.
- Keeping a database identifier when the clone is meant to be inserted as a new record.
- Using clone as a substitute for a transaction or consistent database snapshot.

## Senior Engineer Thinking

Use `clone` when the domain has a meaningful “branch this object” operation. Make `__clone()` enforce ownership and identity rules. If the copy policy is complex, a named factory such as `copyForRevision()` may communicate intent better than a language operator. Prefer immutable values where possible; they eliminate many aliasing questions.

## Exercises

1. Clone a mutable reservation draft and write tests for copied notes, reset identity, and shared versus copied child objects.
2. Create a class containing a mutable `DateTime` and a `DateTimeImmutable`; explain which must be cloned.
3. Design a `copyForRetry()` factory for a command and compare it with `clone` in terms of hidden state.

## Review Questions

1. What does assignment do to an object variable?
2. What does PHP's default clone operation copy?
3. When should `__clone()` clone a nested object?
4. Why is deep cloning not universally correct?
5. Why can cloning not provide database snapshot isolation?

## Summary

`clone` creates a new outer object but starts with a shallow property copy. Use `__clone()` to repair owned mutable state and reset identity, while deliberately sharing immutable values and external identities. Treat cloning as an ownership and lifecycle decision, not a generic deep-copy button.

## Official References

- [PHP Manual: Object Cloning](https://www.php.net/manual/en/language.oop5.cloning.php)
- [PHP Manual: Objects and references](https://www.php.net/manual/en/language.oop5.references.php)
- [PHP Manual: Properties and readonly cloning](https://www.php.net/manual/en/language.oop5.properties.php)
