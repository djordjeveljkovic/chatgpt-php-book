---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 15
title: Functions
slug: functions
status: complete
summary: ../../_ai/chapter-summaries/015-functions-summary.md
---

# Chapter 15 — Functions

## Why This Matters

A function is a named boundary around behavior. It can reduce duplication, name a rule, hide an algorithm, and define the inputs and outputs that callers are allowed to use. It can also become a dumping ground for hidden dependencies, global state, I/O, authorization, and five unrelated reasons to change.

The useful question is not “How small should this function be?” It is:

```text
What decision or transformation does this function own?
What must the caller provide?
What does it return or change?
What can fail?
Which side effects are visible and testable?
```

In PHP, functions range from pure arithmetic helpers to methods that coordinate transactions and external APIs. A production function needs a contract appropriate to its role, not an arbitrary rule that every function must be pure or every operation must be a class.

## Mental Model

Conceptually, a function maps inputs to an outcome:

```text
(arguments, dependencies, state)
            ↓
          function
            ↓
(return value, exception, side effect)
```

A pure function’s result depends only on its arguments and has no externally visible mutation. An application function may deliberately use a clock, database, logger, or queue. The engineering requirement is to make those dependencies and effects visible instead of allowing them to arrive through globals or hidden superglobals.

```php
function overlap(
    DateTimeImmutable $leftStart,
    DateTimeImmutable $leftEnd,
    DateTimeImmutable $rightStart,
    DateTimeImmutable $rightEnd,
): bool {
    return $leftStart < $rightEnd && $rightStart < $leftEnd;
}
```

This function is easy to reason about and test. It says nothing about whether a reservation can actually be inserted; that belongs to a service and a shared database invariant.

## Core Concept

### Declare an honest signature

Use parameter and return types when they express the contract:

```php
function courtLabel(int $number, string $surface): string
{
    return sprintf('Court %d (%s)', $number, $surface);
}
```

Types reject some invalid representations, but they do not validate domain meaning. A positive court number and an allow-listed surface still need a value object or validation rule. Chapter 10 covered this distinction, and Chapter 12 explained strict scalar calls.

Use `void` when a function intentionally has no result. Use `never` only when the function cannot return, such as a function that always throws or terminates. Do not use `mixed` or a broad union to avoid deciding what the function actually returns.

### Parameters are part of an API

Parameters are passed by value by default. Objects give the function a handle to the same object, so mutating an object is observable to the caller; adding `&` is not required for that behavior. Use a reference parameter only when changing the caller’s variable is deliberately the API:

```php
function normalizeInPlace(string &$value): void
{
    $value = strtolower(trim($value));
}
```

Returning the result is commonly clearer:

```php
function normalizedSurface(string $value): string
{
    return strtolower(trim($value));
}
```

Default parameters express defaults for omitted arguments, not validation for explicitly supplied `null`:

```php
function pageSize(int $limit = 50): int
{
    return min(max($limit, 1), 100);
}
```

Keep required parameters before optional ones. In modern PHP, declaring an optional parameter before a required parameter is deprecated or treated as effectively required in cases that cannot be skipped. Prefer `?Type $value` when nullability is part of the contract rather than relying on an old implicit-nullable form.

### Named arguments improve call-site meaning, but names become API

PHP 8.0 introduced named arguments:

```php
$page = paginate(
    query: $query,
    limit: 50,
    cursor: $cursor,
);
```

They can skip defaults and make long calls readable. Parameter names are now part of the call contract, so renaming a public parameter can be a compatibility break even when its position and type stay the same. Do not use names as an excuse to expose a function with twenty optional settings; a configuration object may be clearer.

Positional arguments must precede named arguments, and a parameter must not be supplied twice. Validate public function names and parameter names as API surface during review.

### Variadics and unpacking express repetition

`...` in a declaration collects trailing arguments into an array:

```php
function total(int ...$amounts): int
{
    return array_sum($amounts);
}

$total = total(100, 250, 75);
```

In a call, it unpacks an array or traversable value:

```php
$amounts = [100, 250, 75];
$total = total(...$amounts);
```

Unpacking a large iterable can materialize or process many values at a boundary. Do not use variadics to hide unbounded input; cap batch sizes and validate each value.

### Functions can return behavior

Anonymous functions are `Closure` objects. Arrow functions are concise closures whose used variables are automatically captured by value:

```php
$taxRate = 0.2;
$withTax = fn (int $cents): int => (int) round($cents * (1 + $taxRate));

echo $withTax(1_000); // 1200
```

Traditional closures explicitly import variables and can capture by value or reference:

```php
$threshold = 1_000;

$isLarge = function (int $cents) use ($threshold): bool {
    return $cents >= $threshold;
};
```

Capturing by reference creates a live connection to mutable outer state. Prefer a parameter or value capture when the callback should be deterministic. A closure is not automatically serializable; do not put it in a queue payload or cache entry unless the chosen system has a safe, explicit representation.

PHP 8.1 introduced first-class callable syntax:

```php
$formatter = CourtLabelFormatter::format(...);
$labels = array_map($formatter, $courts);
```

It creates a `Closure` from a callable and preserves the scope where it was acquired. This is generally more analyzable than a string method name. The syntax is version-sensitive, so a library targeting older PHP needs a compatible callable form.

### Separate calculation from orchestration

A function that calculates a conflict and a function that persists a reservation have different contracts:

```php
function reservationConflicts(Reservation $candidate, iterable $existing): bool
{
    foreach ($existing as $reservation) {
        if (overlap(
            $candidate->startsAt,
            $candidate->endsAt,
            $reservation->startsAt,
            $reservation->endsAt,
        )) {
            return true;
        }
    }

    return false;
}
```

This local algorithm is O(n) in the number of existing reservations and O(1) additional space. An orchestration function may call a repository, authorization service, and transaction. Do not pretend that a pure-looking `canReserve()` is sufficient when the durable invariant is shared by concurrent processes.

## How It Works

When PHP calls a function, it evaluates argument expressions, creates a call frame, binds parameters, executes the body, and checks the declared return. The Zend VM manages the frame and the values; exceptions unwind frames until a compatible handler is found. A function call is therefore a runtime boundary, not merely a textual macro substitution.

A function declaration is available in the relevant namespace according to PHP’s name-resolution rules. Defining a function inside a conditional can make availability depend on execution order and is usually a poor bootstrap design. Namespaced functions and autoloaded classes are also different mechanisms: Composer autoloading loads classes on demand, but ordinary function availability should be designed explicitly.

## What PHP Does

PHP supports named functions, methods, closures, arrow functions, generators, and invokable objects. A callable can be a function name, closure, array callable, first-class callable, or object implementing `__invoke()`—subject to visibility and version rules.

Generators change the execution contract:

```php
/** @return Generator<int, Reservation> */
function reservations(iterable $rows): Generator
{
    foreach ($rows as $row) {
        yield Reservation::fromRow($row);
    }
}
```

Calling this function returns a generator; the body runs as it is iterated. `yield` can reduce peak memory for sequential work, but it does not automatically make a database cursor safe to hold indefinitely or make a blocking operation non-blocking. A generator also represents suspended state that must be closed or exhausted according to the resource’s lifetime.

## What Zend Does

The compiler records function metadata and executable operations. A call creates a frame containing parameters, local variables, return state, and exception context. Internal functions cross into extension code, while userland functions execute through the VM. OPcache can cache compiled code, but it does not remove the semantic boundary or make hidden dependencies safe.

Closures carry their executable body and captured state. Generators carry suspended execution state. This is why a callback can retain a large object graph or a request-specific service longer than expected in a worker. Ownership and lifetime remain application concerns even when PHP manages the underlying memory.

## Minimal Example

```php
<?php

declare(strict_types=1);

function addTax(int $priceCents, int $taxBasisPoints): int
{
    if ($priceCents < 0 || $taxBasisPoints < 0) {
        throw new InvalidArgumentException('Amounts cannot be negative.');
    }

    return intdiv($priceCents * (10_000 + $taxBasisPoints), 10_000);
}

echo addTax(1_000, 2_000), PHP_EOL;
```

The function owns one calculation, names its units, validates its preconditions, and returns a value. The rounding policy is truncation and should be documented or changed if the business rule requires another policy.

## Practical Example

A function boundary can make a use case’s dependencies explicit:

```php
function confirmReservation(
    ReservationRepository $reservations,
    Authorization $authorization,
    Clock $clock,
    UserId $actor,
    ReservationId $reservationId,
): void {
    $reservation = $reservations->get($reservationId);

    if (!$authorization->canConfirm($actor, $reservation)) {
        throw new AccessDeniedException();
    }

    $reservation->confirm($clock->now());
    $reservations->save($reservation);
}
```

This is application orchestration, not a pure helper. It is testable because the clock and collaborators are parameters. It still needs a transaction if the state transition and related writes must be atomic, and it needs an idempotency policy if the command can be retried.

## Production Example

A boundary function can adapt a stream into bounded batches without loading the entire input:

```php
/** @return Generator<int, list<string>> */
function batches(iterable $lines, int $size): Generator
{
    if ($size < 1) {
        throw new InvalidArgumentException('Batch size must be positive.');
    }

    $batch = [];

    foreach ($lines as $line) {
        $batch[] = $line;

        if (count($batch) === $size) {
            yield $batch;
            $batch = [];
        }
    }

    if ($batch !== []) {
        yield $batch;
    }
}
```

The generator’s memory is O(size), excluding the source’s own buffers. A worker using it still needs a limit on total input, per-batch transaction behavior, malformed-line handling, progress metrics, and a restart/resume strategy. A generator controls materialization; it does not provide durability.

## Bad Example

```php
function createReservation(): bool
{
    global $pdo, $currentUser;

    $courtId = $_POST['court_id'];
    $pdo->exec("INSERT INTO reservations (court_id) VALUES ($courtId)");
    sendConfirmationEmail($currentUser['email']);

    return true;
}
```

The name hides input, authorization, persistence, notification, and failure semantics. It cannot be safely called from a queue or tested without global process state. Returning `true` after an email call also does not tell the caller whether the database write, message publication, or response succeeded.

## Better Example

```php
function createReservation(
    ReservationCommand $command,
    ReservationRepository $repository,
    Authorization $authorization,
    UserId $actor,
): ReservationId {
    $authorization->assertCanReserve($actor, $command->courtId);

    return $repository->insertIfAvailable($command);
}
```

The function exposes its input and dependencies and returns a meaningful identifier. The repository’s contract must say what happens on conflict, and an outer application layer can coordinate a transaction and an outbox event. This is better because the responsibility is explicit, not because every function should have this exact shape.

## Edge Cases

- Passing an omitted required argument throws `ArgumentCountError` in modern PHP; passing a value that violates a declaration throws `TypeError`.
- Passing `null` does not use a default parameter value. Declare `?T` when null is accepted.
- Named arguments make parameter names part of the API and cannot be mixed with duplicate or out-of-order arguments.
- A variadic parameter collects values into an array; unbounded unpacking can create resource and validation problems.
- `&` on a parameter changes the caller’s variable binding. It is not needed merely because the argument is an object.
- A closure can capture by value or reference. A reference capture may observe later mutation of the outer variable.
- Arrow functions capture used outer variables by value and cannot use a `use` clause.
- A closure created in a method can carry object scope and may retain `$this` longer than expected.
- First-class callable syntax is available from PHP 8.1; check the deployment version before using it in a library.
- Generators execute lazily. Exceptions may occur during iteration rather than when the generator function is called.
- Recursive functions consume call-stack frames and can fail for deep or attacker-controlled input; iterative designs may be safer.
- A function’s return type does not describe side effects. Document writes, emitted events, cache changes, and external calls.

## Performance

Function calls have overhead, but clarity and correct boundaries generally matter more than micro-optimizing a call. The cost of a function is often dominated by what it calls. A callback invoked n times can make an O(n) transformation expensive if it performs I/O or allocates on every invocation.

Use generators when a sequential consumer can process values incrementally and peak memory matters. Do not assume generators improve total CPU or database latency. Repeatedly crossing abstraction layers can also cost something, but flattening code without measurement commonly creates hidden coupling and makes larger costs harder to see.

Choose the data shape that fits the algorithm. A function that repeatedly searches a list may need an indexed map, an SQL query, or a sorted structure. State the complexity and memory trade-off in the function’s contract when it matters.

## Security

Make security-sensitive inputs and dependencies explicit. A function accepting `UserId` still needs authorization; a function accepting a `string $path` still needs an allow-list and path policy. Never let a user select an arbitrary callable, method name, class, or included file unless the dispatch table is controlled and allow-listed.

Do not log complete arguments by default. Function parameters may include passwords, access tokens, personal data, or uploaded content. Catch and translate expected boundary failures without exposing stack traces or SQL. Keep callbacks from untrusted input out of `array_map()` or dynamic invocation.

## Database Interaction

A function that reads and writes shared data should make transaction ownership explicit. Either the function owns a complete atomic unit or its contract says it must be called inside a transaction; ambiguity leads to partial state.

Avoid a repository method that performs a query for each element passed to a function when the database can perform one set-based operation. Conversely, keep domain-specific transformations in PHP when they cannot be expressed safely or readably in SQL. The function boundary is a good place to state which side owns filtering, uniqueness, and aggregation.

## Concurrency

A function call is synchronous in ordinary PHP, but the application can have many concurrent calls in different requests or workers. Local variables and pure calculations are isolated; database writes, files, caches, and messages are shared boundaries.

If a function may be retried, define whether it is idempotent. A function that “sends an email and returns true” has an ambiguous retry story. Use an idempotency key, an outbox, a durable status transition, or a provider operation that can be safely repeated.

## Testing

Test a function according to its contract:

- pure functions: examples, boundaries, invariants, and property-based tests;
- adapters: malformed inputs, type conversion, and safe error translation;
- orchestration: collaborator interactions and important side-effect ordering;
- repositories: real database constraints, transactions, query shape, and hydration;
- callbacks: captured state, invocation count, exceptions, and lifetime where relevant;
- generators: lazy execution, partial consumption, cleanup, and malformed elements;
- retryable functions: idempotency, duplicate calls, timeouts, and failure classification.

Do not mock every function call. A test that verifies private call order but not the durable result can pass while the application is wrong. Prefer real value objects and focused integration tests at database and external-system boundaries.

## Common Mistakes

- Creating functions with vague names and multiple unrelated side effects.
- Hiding dependencies in globals, superglobals, statics, or service locators.
- Using references to “avoid copies” without measuring or needing alias semantics.
- Returning `bool` for every outcome and losing the reason for failure.
- Making every argument optional to avoid changing callers.
- Renaming public parameter names casually after adopting named arguments.
- Capturing mutable state by reference in a long-lived callback.
- Serializing closures into queue messages or cache entries.
- Assuming a generator provides durability or non-blocking I/O.
- Testing mocks instead of business outcomes and shared-system constraints.

## Senior Engineer Thinking

When designing a function, ask:

1. Is this a calculation, adapter, policy, orchestration step, or side-effect boundary?
2. What is the narrowest honest input and output contract?
3. Which dependency should be explicit, and who owns transaction/lifecycle boundaries?
4. What is the time and space complexity for the expected data volume?
5. What happens when the caller retries after a partial failure?
6. What must be tested with a real database, clock, filesystem, or upstream?

Good functions are not necessarily tiny. They are coherent units whose inputs, effects, errors, ownership, and cost can be understood and tested.

## Exercises

1. Refactor a function that reads request globals and writes SQL into an adapter plus a typed domain function. List the responsibilities moved to each boundary.
2. Implement a pure half-open interval conflict function and prove its behavior with adjacent, containing, and identical intervals.
3. Implement a bounded batch generator and test laziness, partial final batches, invalid sizes, and a source that throws halfway through.
4. Convert a public function with many optional positional parameters into a configuration value object or named-argument API. Identify the compatibility trade-off.
5. Write a test for a retryable reservation command that is called twice with the same idempotency key. Define the durable result that both calls should observe.

## Review Questions

1. What belongs in a function contract besides parameter and return types?
2. Why is an object parameter not the same as a reference parameter?
3. When do named arguments create compatibility concerns?
4. What do variadics and unpacking do?
5. How do closures capture state, and why can that affect lifetime?
6. What does first-class callable syntax provide, and when was it introduced?
7. What changes when a function becomes a generator?
8. Why should transaction ownership be explicit?
9. Why is a pure PHP availability check insufficient for concurrent reservation writes?

## Summary

Functions are behavior boundaries whose contracts include inputs, outputs, errors, side effects, dependencies, lifetime, and cost. Use precise types, explicit dependencies, value returns over unnecessary references, named arguments deliberately, variadics with limits, and closures with attention to captured state. Separate pure rules from orchestration, use generators for bounded streaming when appropriate, and test durable behavior at real database and external-system boundaries.

## Official References

- [PHP Manual: User-defined functions](https://www.php.net/manual/en/functions.user-defined.php)
- [PHP Manual: Function arguments](https://www.php.net/manual/en/functions.arguments.php)
- [PHP Manual: Anonymous functions](https://www.php.net/manual/en/functions.anonymous.php)
- [PHP Manual: First-class callable syntax](https://www.php.net/manual/en/functions.first_class_callable_syntax.php)
- [PHP Manual: Generators](https://www.php.net/manual/en/language.generators.php)
