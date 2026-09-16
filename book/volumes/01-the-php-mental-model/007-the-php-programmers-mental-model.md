---
book: The Complete Modern PHP Engineering Book
volume: 1
volume_title: THE PHP MENTAL MODEL
chapter: 7
title: The PHP Programmer's Mental Model
slug: the-php-programmers-mental-model
status: complete
summary: ../../_ai/chapter-summaries/007-the-php-programmers-mental-model-summary.md
---

# Chapter 7 — The PHP Programmer's Mental Model

## Why This Matters

Learning PHP syntax is enough to make a page work. It is not enough to make a system remain correct when requests overlap, dependencies slow down, data grows, processes restart, or a client retries an operation.

Many production bugs are not caused by an obscure language rule. They come from an incomplete model of the program:

- input crossed a boundary without being treated as untrusted;
- state was assumed to be local when it was shared by several workers;
- a date was treated as a string instead of a value with a time zone and a business meaning;
- a database write succeeded but the response never reached the client;
- a retry repeated a non-idempotent side effect;
- a loop was cheap in a test and expensive at production scale;
- a failure happened, but the logs did not contain enough context to explain it.

An experienced PHP programmer therefore asks more than “does this code run?” The better question is:

> What crosses this boundary, what state changes, what can fail, what does it cost, and how will we know what happened?

This chapter collects that habit into a practical mental model. The earlier chapters established that PHP is a language executed by a runtime inside an application environment. Chapter 4 introduced execution as a lifecycle, Chapter 5 contrasted CLI and web entry points, and Chapter 6 described the request/response path. We now use those ideas to reason about an entire feature rather than an isolated statement.

## Mental Model

Model a feature as a controlled transition across boundaries:

```text
Input
  ↓
Boundary and validation
  ↓
Domain decision
  ↓
State transition
  ↓
External effects
  ↓
Output and observation
```

That sequence is not a requirement that every application have six classes or six layers. It is a set of questions. A small script may implement every step in one function. A larger application may place the steps behind controllers, command handlers, domain objects, repositories, and adapters. The design should follow the problem and its constraints, not the other way around.

For every important operation, examine seven dimensions:

| Dimension | Question | Typical design decision |
| --- | --- | --- |
| Boundary | Where did this value come from, and who consumes the result? | Parse and validate at the edge; keep domain code independent of transport. |
| State | What can change, who owns it, and how long does it live? | Choose request, process, database, cache, or message state deliberately. |
| Data | What does each value mean, and which structure represents it? | Use explicit types and representations that match the invariant. |
| Time | Which clock, time zone, ordering, or deadline matters? | Inject the clock; store instants and model intervals intentionally. |
| Failure | What happens when each step fails or is repeated? | Define retries, recovery, consistency, and idempotency. |
| Cost | What resources grow with the input or traffic? | Consider CPU, memory, database, network, latency, and operational work. |
| Observability | What evidence will explain success or failure later? | Emit useful structured logs, metrics, traces, and correlation identifiers. |

These dimensions are connected. A database boundary creates network latency and partial failure. A choice of data structure affects memory and serialization cost. A retry policy changes the required idempotency rules. Observability is not a separate decoration added after the code is finished; it is how the behavior of those decisions becomes inspectable.

## Core Concept

### Boundaries are where assumptions change

A boundary is a place where data, control, trust, or lifetime changes. Examples include:

- an HTTP request entering the application;
- a CLI argument entering a command;
- a message being read from a queue;
- a call from PHP to a database or external API;
- a value crossing from one module into another;
- a response leaving the process.

At a boundary, do not pass along assumptions that belong only to the caller. Convert an external representation into an internal one, validate the rules that can be checked there, and make the remaining contract explicit.

For example, `$_POST['starts_at']` is an input representation. It is not yet a valid reservation start time. It may be absent, malformed, in the wrong time zone, or outside the caller’s authorization. A controller can turn it into a validated command. The reservation rule should receive a meaningful value rather than know about `$_POST`.

This is the same language/runtime/environment distinction from Chapter 1 applied to application design. The HTTP server and PHP-FPM process are part of the environment. The domain rule should not need to know which web server delivered the request.

### State has an owner and a lifetime

A variable is state, but not all state has the same lifetime or visibility. A useful first classification is:

```text
Local value       → one expression or function call
Request state     → one PHP request
Process state      → one CLI or worker process
Shared state      → database, cache, filesystem, or message broker
External state    → another service's system of record
```

In ordinary web execution, request-local variables usually disappear when the request ends. That is a useful property, not a universal law of PHP. A long-running worker can retain objects, static values, caches, open resources, and accidental references across messages. Multiple PHP-FPM workers can process requests concurrently while sharing a database or cache. Chapter 5’s distinction between CLI and web PHP is therefore also a distinction about state lifetime.

For each piece of state, ask:

1. Who is allowed to change it?
2. Who must see the change?
3. How long must it survive?
4. What happens if two actors change it at once?
5. What is the recovery path after a process dies?

If the answer is “the PHP variable remembers it,” the design is probably relying on a lifetime that has not been made explicit.

### Data is a contract, not just a container

The shape of data affects correctness. A string such as `"2026-09-14 19:00"` does not say whether it is UTC, local time, a display value, or a value supplied by an untrusted client. An integer may represent cents, seconds, an identifier, or a count. A PHP array may be a list, a map, a record, or an accidental mixture of all three.

Choose a representation that makes the important invariant visible. A reservation is not merely two strings; it is an interval associated with a court and a user. An amount is not merely a floating-point number; monetary precision and currency matter. An identifier is not interchangeable with a display name just because both can be held in a string.

This idea prepares the reader for Volume II’s treatment of variables, types, arrays, and strict typing. Type declarations are useful, but a declared `string` still does not tell us whether the string is safe, normalized, or semantically correct. Types support a contract; they do not replace one.

### Time is a dependency

Time affects data, behavior, and tests. “Now” is an input to a program even when it is not written as a function parameter. Expiration, scheduling, rate limiting, reservation windows, cache freshness, and timeout decisions all depend on it.

Treat time deliberately:

- distinguish an instant from a calendar date and a local wall-clock time;
- choose and document the time zone used for interpretation and display;
- define intervals as open, closed, or half-open;
- define what happens at a boundary such as exactly 19:00;
- use a monotonic deadline concept for elapsed-duration decisions where the platform provides one;
- make the current time controllable in tests.

For a reservation, a half-open interval `[start, end)` is often useful: a reservation ending at 19:00 does not conflict with another beginning at 19:00. The overlap rule is then:

```text
A overlaps B when:

A.start < B.end AND B.start < A.end
```

The important lesson is not the particular convention. It is that the convention is part of the domain model and must not be left to whichever comparison a developer happens to write first.

### Failure is a possible state transition

An operation is not just success or exception. It may be:

```text
not started
→ in progress
→ succeeded
→ failed before the side effect
→ partially completed
→ succeeded but not acknowledged
```

Consider a payment or reservation request. The database may commit a change, then the connection may fail before PHP sends the response. The client sees an error and retries. If the operation creates a second record or charges a second time, the system has converted an infrastructure failure into a correctness failure.

For each side effect, identify its failure window:

- Can the call time out after the remote system accepted it?
- Can the client send the same request twice?
- Can the worker die after the write and before acknowledgment?
- Can a later retry observe stale state?
- Is the operation safe to repeat, or does it require an idempotency key?

The failure-first sequence from SKELETON.md is a useful design tool: happy path, failure, retry, recovery, consistency, and observability.

### Cost is part of correctness at scale

A program that produces the right answer but exhausts memory or misses its deadline is not correct for its operating environment. Estimate cost before optimizing, but estimate more than Big O:

- CPU work inside PHP;
- memory held by values and intermediate collections;
- database scans, joins, sorting, and round trips;
- network latency and response size;
- queue depth and worker capacity;
- operational and maintenance complexity.

`in_array()` over a list may be perfectly reasonable for ten values and a poor choice for ten million repeated lookups. Loading every database row into PHP to count or sort it may be simple code, but it moves work, memory, and data transfer to the application. A database index may reduce a scan, but it also consumes storage and adds write work. There is no “fast” abstraction independent of constraints.

This extends the chain in SKELETON.md: requirements lead to constraints, then data model, data structure, algorithm, PHP implementation, runtime behavior, database interaction, concurrency, testing, and operations.

### Observability is the evidence of behavior

Logs explain individual events. Metrics show aggregate behavior. Traces connect work across boundaries. They answer different questions:

- Which request or job was this?
- Which user or entity was affected, without leaking sensitive data?
- How long did the database or upstream call take?
- How many requests timed out or were retried?
- Did the operation commit, fail, or become ambiguous?

An observability event should carry enough context to reconstruct the decision without recording secrets or unnecessary personal data. “Reservation failed” is rarely enough. A structured event might include an operation name, correlation identifier, court identifier, outcome, duration, and a safe reason category.

Do not confuse observability with printing everything. Unbounded payload logging can create security, storage, and performance problems. The right question is: what evidence would an engineer need during an incident, and what evidence must never be emitted?

## How It Works

Use a reservation request to walk through the model. Suppose the requirement is:

> A user may reserve a court for a positive interval, and two confirmed reservations for the same court must not overlap.

Before writing a class, make the decisions visible:

```text
Boundary  HTTP request or queue message
Data      user ID, court ID, start instant, end instant
Rule      positive half-open interval; same court cannot overlap
State     reservations are durable shared state
Time      interpret input in a documented zone; compare instants
Failure   duplicate request and concurrent reservation are expected cases
Cost      conflict lookup must not scan unbounded data in PHP
Evidence  record outcome, duration, and correlation ID
```

The request path can then be reasoned about as follows:

1. The entry point reads the transport-specific input.
2. It authenticates the caller and validates the shape and permitted range of the input.
3. It constructs a domain command containing meaningful values.
4. The domain rule checks that the interval is valid and that the user is allowed to reserve.
5. A persistence boundary checks and records shared state.
6. The application translates the result into an HTTP response, command output, or message acknowledgment.
7. The operation records safe evidence about the outcome.

The persistence step is not made safe merely by putting a “check then insert” sequence in PHP. Two workers can both check before either inserts. The database transaction, locking strategy, or constraint must protect the invariant at the shared-state boundary. Later volumes develop those mechanisms in detail; this chapter establishes where the responsibility belongs.

## What PHP Does

PHP supplies the language constructs with which these decisions are expressed: functions, objects, type declarations, exceptions, arrays, date/time objects, streams, and extension APIs. It also supplies the process execution model described in earlier chapters.

PHP does not automatically decide:

- whether a string is a valid domain value;
- whether a database write is atomic with another write;
- whether a retry is safe;
- whether an array is an appropriate data structure;
- whether a request should log a particular field;
- whether a cache is authoritative.

Those are application and system design decisions. A framework may provide conventions and infrastructure for them, but a framework’s helper does not remove the underlying boundary, state, time, failure, cost, and observability questions.

## What Zend Does

At the runtime level, the Zend Engine executes the compiled representation of the PHP program, manages values and function calls, propagates exceptions, and interacts with extensions. That machinery gives ordinary code observable costs and behavior.

For example, assigning and transforming a large array can involve runtime memory management; calling a database extension crosses from PHP into an external system; throwing an exception changes control flow through stack unwinding; and a worker process may keep runtime-managed state alive across many messages. The exact internal structures belong to later chapters on zvals, HashTables, opcodes, memory management, and workers.

The useful mental boundary here is:

```text
PHP code expresses the decision.
Zend executes and manages that code.
The environment determines what the decision can affect.
```

When diagnosing a problem, attribute it before changing it. A memory increase might be an application retaining data, a large runtime allocation, a long-lived worker, or a cache with the wrong lifetime. A slow request might be PHP CPU, a database scan, a network round trip, or contention for a worker. “PHP” is too broad a diagnosis.

## Minimal Example

The interval rule can be expressed as a small, deterministic function:

```php
<?php

declare(strict_types=1);

use DateTimeImmutable;

/**
 * Intervals are half-open: [start, end).
 */
function overlaps(
    DateTimeImmutable $firstStart,
    DateTimeImmutable $firstEnd,
    DateTimeImmutable $secondStart,
    DateTimeImmutable $secondEnd,
): bool {
    if ($firstStart >= $firstEnd || $secondStart >= $secondEnd) {
        throw new InvalidArgumentException('An interval must have positive duration.');
    }

    return $firstStart < $secondEnd && $secondStart < $firstEnd;
}
```

This function has a clear boundary: it receives four time values and returns one decision. It does not read HTTP globals, ask the system clock for “now,” query a database, or write a log. That makes its rule easy to test.

It also has a deliberate limitation. It can decide whether two intervals overlap, but it cannot prove that no other worker inserts a conflicting reservation after the function returns. A pure function can express a domain rule; shared-state protection belongs at the persistence boundary.

## Practical Example

A practical reservation service can preserve the same separation without requiring a large architecture:

```php
<?php

declare(strict_types=1);

final readonly class ReservationRequest
{
    public function __construct(
        public int $userId,
        public int $courtId,
        public DateTimeImmutable $startsAt,
        public DateTimeImmutable $endsAt,
        public string $idempotencyKey,
    ) {
    }
}

interface ReservationStore
{
    public function reserve(ReservationRequest $request): string;
}

final class ReservationService
{
    public function __construct(private ReservationStore $store)
    {
    }

    public function reserve(ReservationRequest $request): string
    {
        if ($request->startsAt >= $request->endsAt) {
            throw new InvalidArgumentException('The reservation interval is invalid.');
        }

        return $this->store->reserve($request);
    }
}
```

The service owns the local domain validation. The store owns the durable shared-state operation. The HTTP adapter can construct `ReservationRequest` and translate expected failures into status codes; a CLI adapter can use the same service and format errors for a terminal. Neither adapter changes what an interval means.

The interface is justified here only because the operation crosses a meaningful boundary and needs different implementations in tests and production. An interface added to every class without a boundary would add names and indirection without adding a useful decision point.

## Production Example

Now add the failure that makes the mental model operational:

```text
Client sends reservation with key K
        ↓
PHP validates request
        ↓
Database commits reservation
        ↓
Connection fails before response reaches client
        ↓
Client retries key K
```

The client’s second attempt is not necessarily evidence that the first attempt failed. The application needs a policy for the ambiguous result. A durable idempotency key can associate both attempts with one logical operation. The persistence boundary can return the original result for a repeated key, reject a conflicting reuse, or expose an explicit pending state.

This design also needs observability. A correlation ID can connect the first request, the database outcome, the transport failure, and the retry. A metric can show the rate of duplicate attempts. Without that evidence, an engineer may see only a user report saying “the reservation failed,” even though the database contains a successful reservation.

The same reasoning applies to queue workers. A worker can commit a state change and die before acknowledging a message. At-least-once delivery then makes a duplicate execution normal, so the handler must be idempotent or use a protected state transition. “The job ran once in my test” says nothing about this production boundary.

## Bad Example

This style hides nearly every important decision:

```php
function reserveCourt(): void
{
    $courtId = $_POST['court_id'];
    $start = $_POST['start'];

    if (date('H') < 9) {
        return;
    }

    global $database;

    $rows = $database->query('SELECT * FROM reservations')->fetchAll();

    foreach ($rows as $row) {
        // Compare loosely formatted strings here.
    }

    $database->exec('INSERT INTO reservations ...');
    echo 'Reserved';
}
```

The problem is not merely style. The function mixes transport parsing, authorization assumptions, system time, shared mutable configuration, an unbounded data load, an unspecified interval rule, persistence, and presentation. It also offers no clear behavior for malformed input, duplicate requests, concurrent inserts, database errors, or a failed response.

## Better Example

Keep the entry point thin and make the decisions explicit:

```php
function handleReservation(array $input, ReservationService $service): string
{
    $request = ReservationInput::fromHttp($input);
    $reservationId = $service->reserve($request);

    return json_encode(
        ['reservation_id' => $reservationId],
        JSON_THROW_ON_ERROR,
    );
}
```

The adapter still needs authentication, authorization, error mapping, and response headers. Those details have not disappeared; they have been placed at the boundary where they belong. The service can be exercised without manufacturing `$_POST`, and the store can use a transaction and database constraint appropriate to the schema.

The better design is not “more classes is better.” Its improvement is that each important assumption has an owner and can be tested or changed independently.

## Edge Cases

The mental model becomes most useful at the edges:

- An empty string, missing field, `null`, and zero may be different inputs. Do not let implicit conversion silently choose domain meaning.
- A daylight-saving transition can make a local time ambiguous or nonexistent. Convert and compare instants deliberately.
- Two adjacent half-open intervals may be valid while two closed intervals would overlap. Document the convention.
- A request can be canceled after the server has started work. Cancellation of the connection does not necessarily undo a committed side effect.
- A cache can contain stale state. Decide whether stale data is acceptable for this operation and what invalidates it.
- A worker can process a message twice, restart midway through a batch, or retain process state from an earlier message.
- A database can succeed while the network response fails, or fail after the server has accepted a request.
- A dataset that fits comfortably in memory during development may not fit when multiplied by traffic or tenant count.

The response to an edge case should be a contract or policy, not an afterthought hidden in a catch block.

## Performance

Performance reasoning starts with the path and its boundaries. For a reservation lookup, compare these approaches:

```text
Load every reservation into PHP
→ scan and compare intervals
→ insert a result
```

and:

```text
Ask the database for a conflict using an indexed, bounded query
→ let the database protect the write invariant
```

The second may be preferable when the database is the owner of reservations and can filter close to the data. It is not automatically superior: query shape, indexes, cardinality, lock contention, and the database engine matter. Measure the actual plan and latency.

For every feature, write down the likely cost shape:

```text
PHP CPU:             O(n) over the input, or repeated O(n) scans?
PHP memory:          Does it materialize the whole result?
Database:            Index lookup, scan, sort, join, or lock wait?
Network:             How many round trips and how much payload?
Latency:             Which dependency determines the deadline?
Operations:          How many services, dashboards, alerts, and failure modes?
```

This is also where the runtime matters. A large temporary array may involve more allocation and copying than its source code suggests; a long-running process may accumulate memory that a short request would release at the end of its lifecycle. Volume IV and Volume V will make these runtime costs concrete.

## Security

Every input boundary is a trust boundary until proven otherwise. Validate shape and range, authenticate the actor, authorize the requested resource, and use parameterized database operations. Do not use a valid PHP type as evidence that a caller is allowed to perform an action.

Failure and observability have security implications too:

- avoid putting passwords, tokens, payment data, or sensitive personal data in logs;
- prevent an idempotency key from being reused across users or operations without a policy;
- treat serialized, cached, and queued data as security-sensitive input;
- avoid returning internal exception details in an HTTP response;
- make retries respect authorization at the time of the retry, not only the first attempt.

The security boundary may not match the code boundary. A controller is not the only place a rule matters if the same operation is reachable from a job, command, or internal API.

## Database Interaction

The database is often the authority for durable shared state. PHP should calculate and coordinate, but it should not pretend that an in-memory check can enforce a rule across processes.

Ask where each invariant belongs:

- PHP can express domain validation and workflow decisions.
- The database can enforce uniqueness, referential integrity, transactions, atomic updates, and some concurrency rules.
- An external service may own facts such as payment status or delivery status.

Pushing every operation into SQL can make domain logic opaque or database-specific. Keeping every operation in PHP can cause unnecessary data transfer, memory use, and race conditions. The right boundary follows ownership and workload.

## Concurrency

“PHP runs one request at a time” can be a useful description of one execution thread, but it does not mean the application has no concurrency. Multiple PHP-FPM workers, CLI processes, queue consumers, scheduled commands, and users can act on shared state at the same time.

The classic danger is check-then-act:

```text
Worker A: check that a court is free
Worker B: check that the court is free
Worker A: create reservation
Worker B: create reservation
```

The fix is not always a distributed lock. Depending on the invariant and database, it may be a unique constraint, transaction with an appropriate lock, atomic update, version check, or a serialized command. Choose the smallest mechanism that protects the required state transition, and test the behavior under contention.

## Testing

Test the contract at the boundary where it is meaningful:

- Test the pure interval rule with adjacent, nested, equal, invalid, and time-zone-normalized values.
- Test input conversion with missing, malformed, unauthorized, and boundary values.
- Test the persistence integration against a real database for transaction, constraint, and query behavior that mocks cannot reproduce.
- Test duplicate idempotency keys and the ambiguous-success policy.
- Test dependency timeouts and failures at the adapter or integration boundary.
- Test the observable outcome that matters: state, response, emitted message, or recorded result.

Injecting a clock or accepting a time value as input makes expiration and scheduling tests deterministic. A test that calls the real current time may pass at 11:59 and fail at 12:00 without any code change.

The goal is confidence in behavior, not a test for every private method. The testing volume will classify unit, integration, feature, API, contract, and end-to-end tests more fully.

## Common Mistakes

- Treating HTTP input, database rows, and domain objects as interchangeable data.
- Assuming request-local state survives into the next request or that worker state disappears after one job.
- Calling the system clock from deep domain code and then struggling to test time-dependent rules.
- Representing money, time intervals, identifiers, and statuses as unexamined strings or floats.
- Performing a check in PHP and assuming it is an atomic guarantee for all workers.
- Retrying every exception without asking whether the side effect is repeatable.
- Loading a whole dataset into PHP because the first dataset was small.
- Adding abstractions without a boundary, or avoiding all abstractions despite a real external boundary.
- Logging a message without an operation ID, outcome, duration, or safe context.
- Treating a successful local test as evidence that a database, network, queue, or process cannot fail.

## Senior Engineer Thinking

When approaching a new feature, make a short design pass before choosing classes:

```text
1. What is the requirement and what must remain true?
2. Which boundaries carry input, output, trust, and lifetime changes?
3. Where does the authoritative state live?
4. What data and time representations make the invariant explicit?
5. What happens on timeout, crash, duplicate delivery, and retry?
6. What is the cost at the expected and worst credible scale?
7. What evidence will show the outcome in production?
```

These questions do not produce one universal architecture. They produce a design whose trade-offs can be explained. A small command may need only a few functions and a transaction. A high-volume API may need queues, idempotency records, rate limits, and tracing. Both can be well engineered if their boundaries and failure behavior match their constraints.

The progression through this book follows the same model. Volume II explains the language values used inside boundaries. Volumes III–V explain objects and the runtime that gives those values cost and lifetime. Volumes VI–VIII develop data structures, Composer, and databases. Later volumes add HTTP, security, testing, architecture, performance, distributed systems, operations, and legacy migration. The chapters are different subjects, but the questions remain stable.

## Exercises

1. Choose a feature you have built, such as a password reset or file upload. Identify its input and output boundaries, its authoritative state, its time dependencies, and one partial-failure scenario.
2. Implement the half-open interval overlap rule and write tests for adjacent, overlapping, nested, equal, and invalid intervals.
3. Take the bad reservation example and list every hidden dependency. Move one dependency at a time behind an explicit input or boundary.
4. For an operation that sends an email after a database write, draw the states that can result if the process dies before and after each step. Decide which state is recoverable and how it will be observed.

## Review Questions

1. Why is a PHP type declaration not enough to establish a domain contract?
2. What is the difference between request state, process state, and shared durable state?
3. Why is “check then insert” unsafe when multiple workers share a database?
4. When might the database be a better place to filter or count than PHP?
5. What information would you want in a log for an operation whose response failed after its database write succeeded?
6. Name a retry that is safe to repeat and one that requires idempotency or another protection. Explain the difference.

## Summary

The PHP programmer’s mental model is a way to reason about behavior across the whole system:

```text
boundaries → state → data → time → failure → cost → observability
```

Boundaries define where assumptions and trust change. State needs an owner and a lifetime. Data representations should make domain invariants visible. Time should be treated as an explicit dependency. Failure includes partial success, retries, crashes, and ambiguous outcomes. Cost includes PHP, runtime, database, network, latency, and operational work. Observability supplies the evidence needed to understand what happened.

PHP and the Zend Engine provide the language and execution machinery, but they do not decide the ownership of a reservation, the safety of a retry, or the authority of a database constraint. Those decisions belong to the design of the application and its surrounding systems.

If the reader carries one habit into the next volume, it should be this: before asking which PHP construct to use, describe the boundary, state transition, data, time, failure policy, cost, and evidence that the construct must support.

## Sources and Further Reading

- [PHP Manual: Language Reference](https://www.php.net/manual/en/langref.php)
- [PHP Manual: Date and Time](https://www.php.net/manual/en/book.datetime.php)
- [PHP Manual: Error Handling](https://www.php.net/manual/en/book.errorfunc.php)
- [PHP Manual: PDO Transactions](https://www.php.net/manual/en/pdo.transactions.php)
