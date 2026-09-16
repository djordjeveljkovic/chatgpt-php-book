---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 22
title: Objects
slug: objects
status: complete
summary: ../../_ai/chapter-summaries/022-objects-summary.md
---

# Chapter 22 — Objects

## Why This Matters

Most production bugs attributed to “objects” are really bugs about identity and ownership. Two variables can refer to the same object, so a change through one variable is visible through the other. Two objects can contain the same values and still be different entities. A function can mutate an object without an `&` parameter, while scalar assignment follows a different mental model.

If you do not reason about that difference, a cache entry can be changed by an unrelated caller, a test can leak state between cases, or a copied-looking object can update the original reservation. This chapter makes object identity concrete before later chapters add properties, methods, and inheritance.

## Mental Model

An object variable is a handle to an object instance. Assigning the variable copies the handle, not a deep copy of the object:

```php
final class Counter
{
    public int $value = 0;
}

$first = new Counter();
$second = $first;
$second->value++;

assert($first->value === 1);
```

`$first` and `$second` are separate variables, but both identify the same `Counter` instance. This is often described casually as “objects are passed by reference.” More precisely, PHP copies an object identifier; the copied identifiers point to the same object. A PHP reference created with `&` is a different language feature and is not needed for ordinary object sharing.

## Core Concept

Identity and equality answer different questions:

```php
final class Player
{
    public function __construct(public int $id, public string $name)
    {
    }
}

$a = new Player(7, 'Mina');
$b = new Player(7, 'Mina');
$c = $a;

assert($a == $b);       // Same class and equal visible state.
assert($a !== $b);      // Different instances.
assert($a === $c);      // Same instance.
```

`==` performs a value-oriented object comparison: instances of the same class are compared by their properties using loose comparison. `===` asks whether the operands refer to the same instance of the same class. Neither operator automatically expresses domain equality. For a player, identity might be the database id; for a money value, equality might be currency plus amount. Make that rule an explicit method or value-object contract when it matters.

## How It Works

When `new` creates an object, the runtime allocates an instance associated with a class entry and returns an object value that variables can hold. A property write follows that instance identity. Passing an object to a function copies the object handle into the parameter:

```php
function rename(Player $player): void
{
    $player->name = 'Renamed';
}

$player = new Player(7, 'Mina');
rename($player);
assert($player->name === 'Renamed');
```

The parameter is not an alias to the variable `$player`; rebinding `$player` inside the function would not rebind the caller’s variable. Mutating the object reached through that parameter does affect the caller’s object:

```php
function replace(Player $player): void
{
    $player = new Player(8, 'Other');
}

replace($player);
assert($player->id === 7);
```

Use `&` only when you deliberately need variable aliasing. It changes the contract and should be visible in the function signature.

## What PHP Does

Objects are not copied by ordinary assignment, argument passing, or return. Returning an object returns another handle to the same instance:

```php
function currentPlayer(Player $player): Player
{
    return $player;
}

$same = currentPlayer($player);
assert($same === $player);
```

The `clone` keyword explicitly creates a new instance. Cloning is shallow by default: object-valued properties in the clone still refer to the same nested objects. Chapter 38 covers `__clone()` and deep-copy decisions. Serialization is not a safe general-purpose deep-copy mechanism and has security and compatibility consequences; it is covered in Chapter 39.

Objects can become unreachable when no variable or property points to them. PHP can reclaim ordinary unreachable cycles through its garbage collector, but you should not use object lifetime as a substitute for explicit transaction, connection, or resource management. A request boundary and a worker shutdown boundary are operational boundaries; they are not business workflows.

## Minimal Example

Make aliasing visible with a method rather than a public property:

```php
final class Score
{
    public function __construct(private int $points = 0)
    {
    }

    public function add(int $points): void
    {
        if ($points < 0) {
            throw new InvalidArgumentException('Points cannot be negative.');
        }

        $this->points += $points;
    }

    public function points(): int
    {
        return $this->points;
    }
}

$original = new Score();
$alias = $original;
$alias->add(3);

assert($original->points() === 3);
```

The method gives the class a place to protect the rule “points never decrease through this operation.” It does not prevent all possible state changes yet; visibility and property design are addressed in Chapters 23–25.

## Practical Example

A reservation may have identity independent of its current status:

```php
final class Reservation
{
    public function __construct(
        public readonly string $id,
        public string $status,
    ) {
    }
}

$reservation = new Reservation('r-100', 'pending');
$sameReservation = $reservation;
$sameReservation->status = 'confirmed';

assert($reservation->status === 'confirmed');
assert($reservation->id === 'r-100');
```

This is a useful model for an entity: the id identifies the same business object while mutable state changes over time. In a real domain, exposing `status` publicly may be too permissive. A `confirm()` method can enforce legal transitions, and a repository can reconstitute state from storage. The example isolates identity before adding those rules.

## Production Example

Shared mutable objects are especially risky when a service caches them:

```php
final class CourtCatalog
{
    /** @var array<int, Court> */
    private array $courts = [];

    public function put(Court $court): void
    {
        $this->courts[$court->id()] = $court;
    }

    public function find(int $id): ?Court
    {
        return $this->courts[$id] ?? null;
    }
}
```

If `find()` returns a mutable object and a caller changes it, the catalog has been changed as a side effect. Options include immutable objects, a command method with validation, a copy/clone with a clearly documented depth, or returning a read model rather than the mutable entity. Choose based on ownership: who is allowed to change the object, and when should the change become durable?

In a normal request, this cache is request-local unless deliberately stored elsewhere. In a long-running worker, it can retain objects across messages and expose stale state or grow without bound. Put an eviction policy and reset boundary around process-local caches.

## Bad Example

This function appears to copy a reservation but does not:

```php
function draftFrom(Reservation $source): Reservation
{
    $draft = $source;
    $draft->status = 'draft';
    return $draft;
}
```

The source reservation is now `draft`. That may corrupt a confirmed booking.

## Better Example

Make the copy decision explicit. For a small value-like object, a named method can construct a separate instance:

```php
final class ReservationDraft
{
    public function __construct(
        public readonly string $courtId,
        public readonly DateTimeImmutable $startsAt,
        public readonly DateTimeImmutable $endsAt,
    ) {
    }

    public static function from(ReservationRequest $request): self
    {
        return new self(
            (string) $request->courtId,
            $request->startsAt,
            $request->endsAt,
        );
    }
}
```

The method documents that a draft is a new value with copied scalar/immutable inputs. If the object graph contains mutable children, decide separately whether those children are shared, copied, or reloaded. Do not promise “deep copy” when you only copied the outer object.

## Edge Cases

- `null` is not an object. Use `?Court` when absence is valid and handle it explicitly.
- A method returning `self` returns the declaring class; `static` can preserve late-static-binding behavior in inheritance. Use either intentionally.
- `isset($object->property)` and `property_exists()` answer different questions, especially for null and inaccessible properties.
- `DateTimeImmutable` returns a new object from modifying operations; `DateTime` mutates itself. The choice affects aliasing.
- A `WeakReference` can observe an object without keeping it alive, but it is an advanced cache/lifecycle tool, not a general ownership model.

## Performance

The cost of an object is not just the class declaration. Each instance has object metadata and property storage, and each nested object adds another allocation. For large collections, measure object-heavy and scalar representations with realistic access patterns. Do not optimize by making every property public or by sharing mutable instances without an ownership rule.

The expensive bug is often accidental work: hydrating the same rows into duplicate graphs, retaining every message in a worker, or triggering lazy I/O from a getter. Object identity is a performance concern because it determines how much state is retained and how many copies or database reads are needed.

## Security

Never treat object identity as authorization. An object id supplied by a client must still be looked up in the caller’s authorization scope. Do not let a caller select an arbitrary class to instantiate, and do not deserialize untrusted object graphs. Keep sensitive properties out of logs and debug dumps.

## Testing

Write tests that prove aliasing and identity deliberately:

```php
function test_aliases_share_an_object(): void
{
    $left = new Score();
    $right = $left;

    $right->add(5);

    assert($left->points() === 5);
    assert($left === $right);
}

function test_replacement_does_not_rebind_the_caller(): void
{
    $player = new Player(7, 'Mina');
    replace($player);

    assert($player->id === 7);
}
```

Also test domain equality separately from identity. If two `ReservationDraft` instances represent the same proposed interval, decide whether your API needs an `equals()` method, value comparison, or no equality promise at all.

## Common Mistakes

- Saying objects are “passed by reference” without distinguishing handles from `&` references.
- Using `==` when the business rule requires identity, or `===` when it requires value equality.
- Assuming assignment or `clone` recursively copies an object graph.
- Returning a mutable cached object without defining ownership.
- Keeping request state in a long-running worker cache forever.
- Using serialization as an unexamined copy or persistence mechanism.

## Senior Engineer Thinking

For every object returned from a method, ask: who owns its state, who may mutate it, and what event makes the mutation durable? For every object passed into a method, ask whether the method observes it, mutates it, or replaces a local handle. For every equality check, name the identity rule in domain language.

These questions prevent a large class of bugs before visibility and constructor design even enter the discussion.

## Exercises

1. Add `equalsById()` to `Reservation` and write tests for two instances with the same id and different status. Explain why `===` is the wrong rule for that method.
2. Create a mutable `Address` nested inside a `Player`. Demonstrate that `clone` is shallow, then choose and implement either a deep clone or an immutable address.
3. Build a request-local `CourtCatalog` and a worker-safe variant. State when each cache is cleared and how stale entries are handled.
4. Pass a `Score` into functions that mutate it, rebind it, and return a new one. Predict the caller’s state before running each test.

## Review Questions

1. What does ordinary object assignment copy?
2. How do `==` and `===` differ for objects?
3. Why does rebinding a function parameter not replace the caller’s variable?
4. Why is a shallow clone insufficient for a graph containing mutable child objects?
5. What object-lifetime problem appears when request-local assumptions are moved into a long-running worker?

## Summary

Objects carry identity and state; variables hold handles to them. Assignment, argument passing, and return copy the handle, so mutations are shared, while rebinding a local variable is not visible to the caller. `===` tests instance identity, `==` performs a limited value comparison, and domain equality should be explicit. Make sharing, copying, ownership, and lifetime deliberate before designing the properties and methods that expose an object.
