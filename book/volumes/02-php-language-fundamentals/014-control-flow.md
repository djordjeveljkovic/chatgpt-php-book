---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 14
title: Control Flow
slug: control-flow
status: complete
summary: ../../_ai/chapter-summaries/014-control-flow-summary.md
---

# Chapter 14 — Control Flow

## Why This Matters

Control flow is how a program chooses, repeats, skips, and exits. The syntax is familiar—`if`, `match`, `foreach`, `while`, `return`, and `throw`—but production correctness depends on what those choices mean at boundaries.

A reservation workflow is not merely a sequence of statements:

```text
parse input
  → reject invalid input
  → check authorization
  → check availability
  → write atomically
  → publish or record the outcome
```

Each branch has a cost and a failure policy. A missing `break` can authorize the wrong path. A loop that queries the database once per row can turn a small feature into an outage. A `continue` can skip cleanup. A retry loop without a deadline can keep a worker occupied forever.

Good control flow makes the normal path visible, handles invalid states deliberately, and gives every exit a clear meaning.

## Mental Model

At the language level, control structures alter which statements execute:

```text
condition → one branch or another
sequence  → next statement
loop      → body, condition, body, condition...
return    → leave the current function
throw     → leave normally executing code and search for a handler
```

At the application level, control flow is a state transition. Ask what has happened before a branch, what side effects happen inside it, and what must be true after it. A condition should not be evaluated in a way that changes the state it is deciding about.

```php
if ($reservation->isCancelable($now)) {
    $reservation->cancel($now);
}
```

The decision and mutation are visible. If `isCancelable()` performs a write or network call, the name is misleading and retry behavior becomes harder to reason about.

## Core Concept

### Sequence first, then guard invalid states

Use guard clauses to reject conditions that make the rest of a function meaningless:

```php
function durationInMinutes(
    DateTimeImmutable $startsAt,
    DateTimeImmutable $endsAt,
): int {
    if ($endsAt <= $startsAt) {
        throw new InvalidArgumentException('End must be after start.');
    }

    return intdiv($endsAt->getTimestamp() - $startsAt->getTimestamp(), 60);
}
```

This is often clearer than wrapping the useful work in several nested `if` blocks. A guard is not a substitute for a domain model; it is a local way to make preconditions explicit.

### `if` chooses based on a boolean context

`if` evaluates its condition in boolean context. Values such as `0`, `'0'`, `''`, `null`, `false`, and an empty array are false-like, but false-like is not the same as invalid or absent. Use explicit comparisons when the domain distinguishes these states:

```php
$rawLimit = $query['limit'] ?? null;

if ($rawLimit === null) {
    $limit = 50;
} elseif (is_string($rawLimit) && ctype_digit($rawLimit)) {
    $limit = (int) $rawLimit;
} else {
    throw new InvalidArgumentException('limit must be an integer.');
}
```

The branch converts the transport value before applying the domain rule. Do not write `if (!$limit)` when zero has a meaningful meaning.

`elseif` and `else if` are equivalent forms, but consistent braces and indentation matter more than the spelling. Always use braces in application code so a future statement cannot silently escape the intended branch.

### `match` is an expression for exact alternatives

Modern PHP’s `match` expression uses strict comparison, returns a value, does not fall through, and must have a matching arm or it throws `UnhandledMatchError`:

```php
enum ReservationStatus: string
{
    case Pending = 'pending';
    case Confirmed = 'confirmed';
    case Cancelled = 'cancelled';
}

function statusLabel(ReservationStatus $status): string
{
    return match ($status) {
        ReservationStatus::Pending => 'Awaiting payment',
        ReservationStatus::Confirmed => 'Confirmed',
        ReservationStatus::Cancelled => 'Cancelled',
    };
}
```

The lack of a default arm is intentional: adding a new enum case forces the mapping to be considered at runtime. Use `default` when the fallback is a real policy and unknown values are safe to group, not merely to silence an exhaustive decision.

`match` is an expression, so it can be assigned or returned. Its arms should remain small. If an arm needs several statements and side effects, extract a named function or make the workflow explicit rather than hiding it in an expression.

### `switch` is useful but has legacy semantics

`switch` compares cases loosely and falls through until `break`, `return`, or the end of the structure:

```php
switch ($legacyCode) {
    case 'paid':
        $state = 'confirmed';
        break;
    case 'void':
        $state = 'cancelled';
        break;
    default:
        throw new InvalidArgumentException('Unknown legacy code.');
}
```

Fall-through can be deliberate for grouped cases, but make it obvious with a comment and tests. In new code, prefer `match` for a value mapping or `if` for a predicate with side effects. Do not mechanically replace every old `switch`: a large command dispatcher may be clearer as a map of handlers or a strategy object.

### Loops need a termination and work model

`foreach` is the usual choice for traversing an array or `Traversable` value:

```php
foreach ($courts as $court) {
    if (!$court->isActive()) {
        continue;
    }

    $activeCourts[] = $court;
}
```

`for` is useful when an index or a known counter is the actual state. `while` is appropriate when the next iteration depends on a changing condition, such as reading a stream. `do ... while` always runs once, so it is dangerous when the input may already be invalid.

Every loop should answer:

- What makes it stop?
- What is the maximum work per iteration?
- What happens if the collection grows without bound?
- Which side effects have occurred if iteration fails halfway?

`break` exits the nearest loop or switch. `continue` skips to the next iteration; in nested loops, `continue 2` targets an outer level. Numeric levels are occasionally useful, but named functions or a result flag often communicate the boundary better.

### `return` and `throw` are different exits

`return` gives a value to the caller and ends the current function. `throw` transfers control to the nearest compatible exception handler after stack unwinding. Do not return `false` for every failure when callers need to distinguish invalid input, not-found, and infrastructure failure.

PHP also supports `throw` as an expression, which is useful for a required fallback:

```php
$court = $courts[$courtId]
    ?? throw new RuntimeException('Court was not loaded.');
```

Use the expression sparingly. A statement is often easier to instrument and annotate when failure has operational significance.

## How It Works

The compiler turns branches and loops into jumps and repeated operations in the executable representation. The Zend VM evaluates the condition, chooses a target, and executes the selected operations. A loop does not create a magical collection snapshot: if the body calls a database, sends a message, or mutates an object, those effects occur each time the path runs.

This model explains why a `break` changes control flow but does not undo work already done. If three messages were published before the fourth iteration failed, catching the exception outside the loop cannot retract those messages automatically. Use a transaction, an outbox, a compensating action, or a resumable workflow according to the boundary.

## What PHP Does

PHP control structures include:

| Construct | Kind | Typical use |
| --- | --- | --- |
| `if` / `elseif` / `else` | Statement | Predicates and guarded workflows |
| `match` | Expression | Exhaustive value mapping |
| `switch` | Statement | Legacy or grouped case dispatch |
| `foreach` | Loop | Arrays and iterables |
| `for` | Loop | Counter/index-controlled repetition |
| `while` | Loop | Condition-controlled repetition |
| `do ... while` | Loop | At-least-once work |
| `break` / `continue` | Loop exits | Stop or skip work |
| `return` | Function exit | Produce a result |
| `require` / `include` | Loading flow | Bootstrap and optional resources |
| `goto` | Jump | Rare low-level control transfer |

Alternative syntax (`if: ... endif;`, `foreach: ... endforeach;`) can be useful in templates, but mixing styles in application logic reduces consistency. `goto` is legal and has narrow uses in generated code or cleanup paths, yet structured control flow and small functions are easier to review in ordinary domain code.

## What Zend Does

Branches and loops are represented as executable operations with jump targets. An exception changes the normal path and unwinds call frames until a handler matches. Generators and fibers add suspended control contexts; they do not make a blocking database call asynchronous by themselves. Those topics belong in later runtime chapters, but the core lesson is the same: the execution context and its side effects outlive a single line of source.

## Minimal Example

```php
<?php

declare(strict_types=1);

$numbers = [2, 4, 7, 8];
$firstOdd = null;

foreach ($numbers as $number) {
    if ($number % 2 !== 0) {
        $firstOdd = $number;
        break;
    }
}

var_dump($firstOdd); // int(7)
```

The loop has a finite input, a clear exit condition, and no hidden I/O. This is the smallest useful model for `foreach` and `break`.

## Practical Example

Process a batch while making rejection and continuation policy explicit:

```php
/** @param list<array<string, mixed>> $rows */
function importRows(array $rows): ImportReport
{
    $accepted = 0;
    $rejected = [];

    foreach ($rows as $index => $row) {
        try {
            $reservation = ReservationInput::fromArray($row);
        } catch (InvalidArgumentException $exception) {
            $rejected[$index] = $exception->getMessage();
            continue;
        }

        saveReservation($reservation);
        $accepted++;
    }

    return new ImportReport($accepted, $rejected);
}
```

This “continue on bad row” policy is appropriate only if each row is independent and partial import is acceptable. If one invalid row means the file is untrustworthy, fail the whole batch and use a transaction. Control flow encodes product policy; it is not only style.

## Production Example

A bounded retry loop needs a deadline, a classification of retryable failures, and observability:

```php
function fetchWithRetries(callable $fetch, int $maxAttempts): string
{
    $maxAttempts = max(1, $maxAttempts);

    for ($attempt = 1; $attempt <= $maxAttempts; $attempt++) {
        try {
            return $fetch();
        } catch (TemporaryUpstreamException $exception) {
            if ($attempt === $maxAttempts) {
                throw $exception;
            }

            $delayMicros = min(500_000, 50_000 * (2 ** ($attempt - 1)));
            usleep($delayMicros);
        }
    }

    throw new LogicException('Retry loop ended unexpectedly.');
}
```

In a real service, the HTTP client also needs connect and total timeouts, jitter, a cancellation/deadline budget, and a request identifier in logs. Retrying a non-idempotent operation can duplicate side effects. The loop should be allowed only when the operation is idempotent or protected by an idempotency key.

## Bad Example

```php
foreach ($_POST['reservations'] as $row) {
    if (!$row['court_id'] || !$row['starts_at']) {
        continue;
    }

    $pdo->exec("INSERT INTO reservations ...");
}
```

This conflates zero-like values with missing data, trusts an unvalidated array shape, silently drops bad rows, interpolates data into SQL, and leaves the transaction policy implicit. A failure after several inserts can produce a partial import that nobody can explain.

## Better Example

```php
$pdo->beginTransaction();

try {
    foreach ($inputRows as $index => $row) {
        $reservation = ReservationInput::fromArray($row);
        $insertReservation($reservation);
    }

    $pdo->commit();
} catch (Throwable $exception) {
    $pdo->rollBack();
    $logger->error('Reservation import rolled back.', [
        'row' => $index ?? null,
        'exception' => $exception,
    ]);
    throw $exception;
}
```

The adapter validates before the persistence operation, the transaction policy is explicit, and failure is observable. The insert should be a prepared statement, and if the input is huge the operation should be chunked with a resumable import design rather than one unbounded transaction.

## Edge Cases

- `if` uses boolean conversion; `0`, `'0'`, `''`, `null`, `false`, and `[]` are false-like but may be valid domain values.
- `match` uses strict comparison, has no fall-through, and throws `UnhandledMatchError` if no arm matches and no default exists.
- `switch` uses loose comparison and can fall through without `break`.
- `continue` inside `switch` behaves like `break`; use `continue 2` when the intent is to continue an enclosing loop.
- `break` and `continue` levels are counted through nested loops and switches. Refactor when numeric levels become difficult to follow.
- `foreach ($items as &$item)` leaves the loop variable bound by reference after the loop. Avoid it or `unset($item)` immediately afterward.
- Modifying an array while iterating it has defined cases and surprising cases; prefer building a result or using a dedicated transformation.
- A `do ... while` body executes once even when the condition is initially false.
- A `return` in a `finally` block can override a pending return or exception. Do not use it for ordinary cleanup.
- A retry loop can amplify an outage and can duplicate a non-idempotent side effect.
- `include` can continue with a warning when a file is missing; `require` fails more strongly. Bootstrap policy should be explicit.
- A loop that waits without a deadline can consume a worker indefinitely.

## Performance

For an in-memory list of `n` items, a single pass is O(n) time and O(1) additional space if the result is streamed or accumulated elsewhere. Nested searches can be O(n²); replacing them with a map keyed by ID can reduce lookup to average O(1) per item at the cost of memory.

The bigger cost may be outside the loop. `foreach ($rows as $row) { SELECT ... }` creates an N+1 database pattern, and retrying a slow upstream call inside many iterations multiplies latency. Batch queries, use indexes, paginate, stream large inputs, and measure the full boundary rather than optimizing loop syntax.

## Security

Control flow is often the authorization flow. Make the deny path explicit and fail closed:

```php
if (!$authorization->canEdit($actor, $reservation)) {
    throw new AccessDeniedException();
}

updateReservation($reservation);
```

Do not rely on a later branch to compensate for an early default allow. Validate before indexing arrays, cap loop counts and uploaded-file work, use timeouts for external calls, and avoid logging sensitive payloads on every retry. A loop processing attacker-controlled input is a resource-exhaustion boundary.

## Database Interaction

Do not use a PHP loop to simulate a database invariant without a transaction:

```php
foreach ($requestedReservations as $reservation) {
    if ($repository->findConflict($reservation) === null) {
        $repository->insert($reservation);
    }
}
```

Each check and insert can race with another request. Prefer a transaction with the appropriate lock or constraint, and let the database filter and aggregate rows where it is efficient. PHP control flow should coordinate the transaction and interpret results, not pretend process-local branches are global serialization.

## Concurrency

A loop in one PHP process is sequential, but many requests and workers can run that loop concurrently. A check-then-act branch is therefore not atomic. Queue consumers may also receive the same message twice, so the loop body must be idempotent or use a durable deduplication key.

Retry control flow deserves special care: exponential backoff reduces pressure, but synchronized workers can still retry in waves. Add jitter, cap the attempts, propagate a deadline, and stop retrying errors that are permanent.

## Testing

Test control-flow outcomes and side effects:

- cover valid, invalid, empty, zero-like, and boundary inputs;
- assert `match` mappings for every enum case and an unknown value where relevant;
- test `switch` grouped cases and deliberate fall-through;
- test `break`/`continue` paths and verify skipped work really is skipped;
- test partial-import versus all-or-nothing policy with a real transaction boundary;
- test retry count, backoff classification, timeout/deadline, and idempotency behavior;
- test authorization denial before the side effect occurs;
- use property-based tests for interval predicates and loop invariants when the state space is broad.

Mutation testing is valuable for high-risk branches: replace `&&` with `||`, remove a guard, or invert an authorization decision and verify that tests fail. Coverage alone does not prove that the branch’s meaning is protected.

## Common Mistakes

- Treating false-like values as universally invalid.
- Omitting braces or relying on indentation to define a branch.
- Forgetting `break` in `switch`.
- Using `default` in every `match` and hiding newly introduced cases.
- Calling the database or network inside an unbounded loop.
- Silently skipping malformed input without a report or metric.
- Retrying non-idempotent work without a key or transaction design.
- Using `continue` in a way that skips required cleanup.
- Returning from `finally` and masking a failure.
- Assuming sequential code inside one worker eliminates cross-request races.

## Senior Engineer Thinking

For every branch or loop, ask:

1. What states are accepted, rejected, or treated as unknown?
2. What side effects have happened at each exit?
3. Is the operation idempotent if the process crashes and retries?
4. What is the termination condition and worst-case work?
5. Does the database or another shared system need to enforce the invariant?
6. Which metric, log, or trace tells us that the unexpected path is occurring?

The best control flow is not necessarily the shortest. It is the one whose state transitions, failure behavior, resource limits, and ownership can be reviewed without simulating hidden work in one’s head.

## Exercises

1. Rewrite a deeply nested reservation validation function using guard clauses. List the preconditions and the side effect that must happen last.
2. Map every case of a backed enum with `match`, deliberately omit one case, and write a test that detects the resulting `UnhandledMatchError`.
3. Build a batch importer with a configurable all-or-nothing or skip-invalid policy. Record rejected row numbers and test a failure halfway through.
4. Implement a bounded retry helper with exponential backoff, jitter, a deadline, and an idempotency note. Test permanent and temporary failures.
5. Measure a loop that performs one query per row, then replace it with a set-based query. Compare query count and total latency.

## Review Questions

1. Why are guard clauses useful?
2. How does `match` differ from `switch`?
3. Why can a `default` arm hide a maintenance problem?
4. What should every unbounded-looking loop define?
5. What is the difference between `return`, `break`, `continue`, and `throw`?
6. Why is “skip invalid rows” a product and consistency decision?
7. How can a retry loop worsen an outage?
8. Why does a sequential loop still have concurrency problems in a web application?
9. What makes a control-flow test stronger than line coverage?

## Summary

Control flow expresses state transitions, not merely syntax. Use explicit guards, strict and exhaustive `match` mappings, deliberate `switch` semantics, bounded loops, and clear exits. Treat side effects, retries, partial work, authorization, database transactions, and concurrent workers as part of the branch design. Test both the selected path and the work that must not occur on rejected or failed paths.

## Official References

- [PHP Manual: Control structures](https://www.php.net/manual/en/language.control-structures.php)
- [PHP Manual: `match`](https://www.php.net/manual/en/control-structures.match.php)
- [PHP Manual: `switch`](https://www.php.net/manual/en/control-structures.switch.php)
- [PHP 8.0 migration guide](https://www.php.net/manual/en/migration80.new-features.php)
