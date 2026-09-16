---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 9
title: Variables
slug: variables
status: complete
summary: ../../_ai/chapter-summaries/009-variables-summary.md
---

# Chapter 9 — Variables

## Why This Matters

A variable is a name through which a PHP program reads or changes a value. That definition is simple enough to hide the engineering questions that matter:

- Is the value local to one function, one request, or one long-running worker?
- Does assigning it make an independent value or an alias?
- Can it be absent, `null`, or present with an empty value?
- Who is allowed to change it?
- Did it come from a trusted program boundary or an untrusted request?
- What memory and lifetime does it have when the value is large?

Many PHP bugs are variable-model bugs. A default is confused with a missing field. A reference unexpectedly changes a caller’s array. A static cache survives longer than expected. A global hides a dependency. A worker retains request data after the message has been acknowledged.

Variables are not a database, a session, or a synchronization mechanism. They are runtime state owned by a scope and a process. Chapter 7 introduced lifetime and ownership; this chapter applies those ideas to PHP’s variable semantics.

## Mental Model

Think of a variable as a named slot in a scope that currently points to a value. The conceptual model is:

```text
scope + name
    → current value
```

The name begins with `$`, is case-sensitive, and must follow PHP’s identifier rules after the dollar sign. Assignment normally gives the destination the value of the expression:

```php
$original = ['status' => 'draft'];
$copy = $original;

$copy['status'] = 'published';

var_dump($original['status']); // draft
var_dump($copy['status']);     // published
```

The conceptual result is independent values. For arrays and scalar values, the engine can delay physical copying until a write requires it; that copy-on-write optimization is explained in Volume IV. Do not replace the language rule “assignment by value” with the inaccurate rule “PHP always copies all bytes immediately.”

Objects are different in a useful way:

```php
final class Counter
{
    public function __construct(public int $value)
    {
    }
}

$first = new Counter(1);
$second = $first;
$second->value = 2;

var_dump($first->value); // 2
```

Both variables hold an object handle for the same object. Assignment did not create an alias between the variable names, but both handles identify the same mutable object. Use `clone` when an independent object is required, and design immutable objects when shared mutation would be surprising.

## Core Concept

### Assignment is not reference binding

Normal assignment:

```php
$a = 10;
$b = $a;
$b = 20;

echo $a; // 10
```

Reference assignment deliberately makes two variable names aliases:

```php
$a = 10;
$b = &$a;
$b = 20;

echo $a; // 20
```

References are a language feature, not a general performance tool. They can be useful when an API explicitly requires an in-place update, but they create a second path to mutate state. Prefer returning a new value or mutating an object with a clear method when that gives the contract a simpler shape.

Function parameters are passed by value by default. An object parameter still gives the function access to the same object, so the function can mutate that object; it does not mean the variable in the caller and parameter are aliases. A parameter declared with `&` is an explicit reference parameter and has stronger coupling:

```php
function normalizeInPlace(string &$value): void
{
    $value = strtolower(trim($value));
}

$surface = ' Clay ';
normalizeInPlace($surface);
```

Use this form only when changing the caller’s variable is part of the API. A return value is often clearer:

```php
function normalizedSurface(string $value): string
{
    return strtolower(trim($value));
}
```

### A variable can be undefined, null, or empty

These states are not interchangeable:

```php
$missing = null;
$emptyString = '';
$zero = 0;
$false = false;

var_dump(isset($missing));     // false: isset() is false for null
var_dump(isset($emptyString)); // true
var_dump(isset($notSet));      // false, without a value to read
```

`isset($value)` asks whether a variable exists and is not `null`. `empty($value)` asks whether a value is considered empty under PHP’s rules, which include values such as `0`, `'0'`, `''`, `false`, `null`, and an empty array. Those rules can be useful at a transport boundary, but they are often too broad for a domain rule.

If the difference between “field omitted” and “field explicitly set to null” matters, use an operation that preserves that distinction:

```php
$payload = ['nickname' => null];

if (array_key_exists('nickname', $payload)) {
    // The key was supplied, even though its value is null.
}
```

Once a value enters the domain, convert ambiguous input states into an explicit model. For example, represent a patch as a command whose field is either “leave unchanged” or “set this value,” rather than passing an arbitrary array through several layers.

### Destructuring is assignment with a shape

Modern PHP can unpack lists and keyed arrays into variables:

```php
[$start, $end] = [new DateTimeImmutable('09:00'), new DateTimeImmutable('10:00')];

['id' => $courtId, 'surface' => $surface] = [
    'id' => 3,
    'surface' => 'clay',
];
```

Destructuring is concise when the shape is established and trusted. At an external boundary, verify that the keys exist and have the expected types before unpacking. A compact assignment does not validate an API payload.

### Scope gives variables a lifetime

Function scope, global scope, static locals, closures, and object properties have different ownership rules. Detailed scope behavior belongs in Chapter 16, but the lifetime distinction is already important:

```php
function nextRequestNumber(): int
{
    static $number = 0;

    return ++$number;
}
```

The static local persists between calls in the same PHP process. In an ordinary web request, that may mean only for the request. In a long-running worker, it can persist across many messages. It is not a cluster-wide counter and is not safe coordination for multiple processes.

A local variable is usually easier to reason about because its owner and lifetime are narrow. If state must outlive the process or be visible to other workers, put it in an explicit shared system such as a database, cache, or queue.

### Superglobals are boundary inputs

PHP provides superglobals such as `$_GET`, `$_POST`, `$_FILES`, `$_COOKIE`, `$_SESSION`, `$_SERVER`, `$_ENV`, and `$argv`. They are convenient access points, not trusted domain objects. Their availability and contents depend on the execution environment.

Read them at the entry point, normalize and validate the values, then pass an explicit value onward:

```php
function requestedCourtId(array $input): int
{
    $raw = $input['court_id'] ?? null;

    if (is_int($raw) && $raw > 0) {
        return $raw;
    }

    if (is_string($raw) && ctype_digit($raw) && (int) $raw > 0) {
        return (int) $raw;
    }

    throw new InvalidArgumentException('court_id must be a positive integer.');
}

$courtId = requestedCourtId($_POST);
```

This function still needs a policy for very large numeric strings and authorization, but it no longer makes every downstream function know about `$_POST`. Chapter 6 and Chapter 7 established why boundary conversion matters.

## How It Works

At a conceptual level, each variable access resolves a name in a scope and reads or writes the associated runtime value. The Zend Engine represents values with internal structures such as zvals and manages storage, references, object handles, and copy-on-write. Those terms describe implementation mechanisms, not additional PHP variables visible in source.

For a non-object value, this sequence is useful:

```text
$a = large array
$b = $a       → same logical value, sharing may be possible
$b[0] = ...   → separation before the write if required
```

For an object:

```text
$a = new Basket
$b = $a       → two handles, one object
$b->add(...)  → the same object is changed
```

For a reference:

```text
$b = &$a      → two names bound to one variable/reference set
```

This model predicts surprising behavior without claiming that source assignment maps one-to-one to immediate memory copies. The exact refcounting and HashTable behavior belongs to Volume IV.

## What PHP Does

PHP supports values that are created and changed dynamically. A variable need not be declared before first assignment, although static analysis and explicit initialization make application code safer. Reading an undefined variable emits a warning in modern PHP and produces `null` in contexts where execution continues; relying on that fallback hides a bug.

PHP also provides variable variables:

```php
$field = 'surface';
$$field = 'clay';
```

This creates or accesses `$surface` dynamically. It can be appropriate for a narrowly controlled language feature, but user input must never be allowed to choose arbitrary variable names. A map keyed by an allow-listed name is easier to inspect and secure:

```php
$allowed = ['surface', 'court_id'];
$field = 'surface';

if (in_array($field, $allowed, true)) {
    $attributes[$field] = $value;
}
```

Constants are another kind of named value and are not variables. A constant expresses configuration or identity that should not be reassigned in the same way. Class constants, enums, and configuration objects each have different ownership and loading concerns; choose among them deliberately rather than using globals for everything.

## What Zend Does

Zend allocates and manages the runtime representation for values, separates shared arrays when a write requires it, and keeps object identity stable while handles are assigned. The engine also maintains references when source code explicitly asks for them. These mechanisms explain why a large array may appear cheap to assign and then consume more memory when mutated, and why an object can change through another variable without an explicit reference operator.

The runtime does not know whether a variable represents “money,” “a user identifier,” or “untrusted HTML.” It knows a value and its type. Semantic constraints belong to the application, and type declarations—covered next in Chapter 10—can express only part of those constraints.

## Minimal Example

This example contrasts scalar assignment, array assignment, object handles, and explicit cloning:

```php
<?php

declare(strict_types=1);

final class Preferences
{
    public function __construct(public string $theme)
    {
    }
}

$settings = ['timezone' => 'UTC'];
$otherSettings = $settings;
$otherSettings['timezone'] = 'Europe/Belgrade';

$preferences = new Preferences('light');
$otherPreferences = $preferences;
$otherPreferences->theme = 'dark';

$independentPreferences = clone $preferences;
$independentPreferences->theme = 'system';

var_dump($settings['timezone']);             // UTC
var_dump($preferences->theme);               // dark
var_dump($independentPreferences->theme);    // system
```

The array mutation separates the logical values. The object mutation affects both handles. `clone` creates a new object, although nested objects inside it may still be shared unless the class implements the appropriate `__clone()` behavior.

## Practical Example

A request patch needs to distinguish omitted fields from fields explicitly set to `null`:

```php
<?php

declare(strict_types=1);

final readonly class ProfilePatch
{
    public function __construct(
        public bool $changesDisplayName,
        public ?string $displayName,
        public bool $changesPhone,
        public ?string $phone,
    ) {
    }

    public static function fromInput(array $input): self
    {
        return new self(
            changesDisplayName: array_key_exists('display_name', $input),
            displayName: self::nullableString($input['display_name'] ?? null),
            changesPhone: array_key_exists('phone', $input),
            phone: self::nullableString($input['phone'] ?? null),
        );
    }

    private static function nullableString(mixed $value): ?string
    {
        if ($value === null) {
            return null;
        }

        if (!is_string($value)) {
            throw new InvalidArgumentException('Profile fields must be strings or null.');
        }

        return trim($value);
    }
}

$patch = ProfilePatch::fromInput($_POST);
```

The value object makes an important variable state explicit. The application can now say “do not change the phone” when `changesPhone` is false, and “clear the phone” when it is true and `phone` is null. Passing raw superglobal data into the repository would lose that clarity.

## Production Example

Long-running workers make variable lifetime operationally visible:

```php
final class ImportWorker
{
    /** @var list<array<string, mixed>> */
    private array $processedRows = [];

    public function handle(array $message): void
    {
        $rows = $this->loadRows($message['file']);
        $this->processedRows = array_merge($this->processedRows, $rows);
    }
}
```

If `$processedRows` is not needed after one message, this worker retains every row for the life of the process. Under a normal request lifecycle that mistake may disappear when the request ends; under a queue worker it becomes a memory leak or at least unbounded retention. Keep message-local values local, clear caches deliberately, batch large inputs, and recycle workers as an operational safety net. Chapter 65 returns to long-running PHP processes in detail.

## Bad Example

This function hides input, state, and ownership:

```php
function updateCourt(): void
{
    global $db, $currentUser;

    $id = $_POST['id'];
    $surface = $_POST['surface'] ?? '';
    $GLOBALS['last_surface'] = $surface;

    $db->exec("UPDATE courts SET surface = '$surface' WHERE id = $id");
}
```

The variables come from an untrusted boundary, the global variables hide dependencies, the value is not validated or authorized, and the SQL is injectable. The global assignment also creates process-wide coupling without defining a useful lifetime. Even if the query succeeds, another caller may observe state it should not own.

## Better Example

Make the boundary and dependency visible, and let a prepared statement carry the values safely:

```php
function updateCourt(PDO $db, int $courtId, string $surface): void
{
    if ($courtId < 1 || !in_array($surface, ['clay', 'hard', 'grass'], true)) {
        throw new InvalidArgumentException('Invalid court data.');
    }

    $statement = $db->prepare(
        'UPDATE courts SET surface = :surface WHERE id = :id',
    );
    $statement->execute([
        'surface' => $surface,
        'id' => $courtId,
    ]);
}
```

The caller still must authenticate and authorize the actor. The function owns a local decision and a database operation; it does not pretend that a variable in PHP is the source of truth for the court.

## Edge Cases

- `isset($value)` is false for an existing variable whose value is `null`. Use `array_key_exists()` when key presence itself matters.
- `empty()` uses broad truthiness rules. It can treat `0` and `'0'` as empty, which is wrong for many identifiers and quantities.
- Reading an undefined variable is a warning in current PHP. Initialize required state or fail at the boundary instead of relying on `null` fallback.
- `unset($value)` removes the variable binding; it does not guarantee that every other reference, object handle, or copy of the value disappears.
- `foreach ($items as &$item)` binds the loop variable by reference. Call `unset($item)` after the loop or, preferably, avoid the reference unless in-place updates are intentional.
- Destructuring a missing key or an unexpected shape can produce warnings or undefined values. Validate external arrays first.
- `clone` is shallow by default. An object containing mutable nested objects may still share those nested objects after cloning.
- A static local and a static property persist for the process lifetime, not across independent PHP processes. In a worker, that can mean many messages.
- Superglobals vary by entry point. `$_POST` is not the source of CLI options, and environment variables can be absent or differently encoded.
- Copy-on-write reduces unnecessary copying but does not make mutation free. Multiple mutated copies of a large array can consume substantial memory.

## Performance

For large values, distinguish logical assignment from physical allocation. Assigning a large array to another variable may initially share storage. A later write can force separation, and repeated mutations can create several large allocations. This is one reason to avoid passing enormous arrays through layers when a stream, iterator, or focused projection is enough.

References can prevent a clean ownership model and make optimization or analysis harder; they should not be introduced solely because copying sounds expensive. Measure allocations and peak memory for the real workload. Often the better performance change is to avoid loading unnecessary rows, use a generator for sequential input, or let the database filter data before PHP receives it.

In a normal request, variables become unreachable at the end of the request and the process can reclaim request state. In a worker, the process remains alive. A cache or accumulator stored in a property, static, global, or closure can therefore turn a per-message cost into a process-growing cost.

## Security

Treat every externally populated variable as untrusted until validated:

- `$_GET`, `$_POST`, cookies, headers, uploaded-file metadata, and environment variables are input boundaries;
- type checks do not establish authorization;
- allow-list fields and operations instead of using user input as a variable name or method name;
- use prepared statements for SQL values and safe APIs for commands and paths;
- do not put tokens, passwords, or sensitive request payloads into debug variables that may be logged;
- clear sensitive values when a long-lived process no longer needs them, while recognizing that language-level clearing is not a complete memory-forensics guarantee.

A variable called `$safeHtml` is not safe because of its name. Safety comes from the encoding, validation, and trust boundary that produced it.

## Database Interaction

Variables are often the bridge between request data and SQL. Bind values rather than building SQL source with interpolation. Also distinguish a missing value from `null` when constructing updates: a patch that omits `surface` should not overwrite the database column with `NULL` merely because an array lookup returned a default.

Do not use a PHP variable as an application-level lock:

```php
if (!isset($locks[$courtId])) {
    $locks[$courtId] = true;
    // This does not coordinate other PHP-FPM workers.
}
```

The variable is local to one process. A database transaction, unique constraint, or shared lock service may be appropriate depending on the invariant. Chapter 35 develops why application checks alone are insufficient.

## Concurrency

PHP variables do not become shared merely because several workers run the same code. Each process has its own ordinary variables, statics, and object graphs. Concurrent requests coordinate only when they use a shared boundary such as a database, cache, filesystem, socket, or message broker.

This gives request-local variables a useful default: one request cannot accidentally mutate another request’s local variable. It does not make all state safe. A long-running worker can accidentally carry state from message A into message B, and shared systems can still observe conflicting writes. Make lifetime and ownership explicit, then test the shared boundary under concurrency.

## Testing

Test variable behavior through contracts rather than implementation trivia:

- verify that a normal array assignment does not let a later mutation change the original logical value;
- verify that an object mutation is visible through another handle when sharing is intended;
- verify `clone` behavior for nested mutable state if cloning is part of the API;
- test omitted, null, empty-string, zero, and false input separately;
- test superglobal adapters with ordinary arrays so tests do not depend on global process state;
- test that a worker clears or bounds message-local accumulators;
- use static analysis to find possibly undefined variables and invalid array shapes.

Avoid tests that merely assert an internal reference count. Such details can change between PHP versions and are not usually the application contract. Test observable state, output, side effects, and failure behavior.

## Common Mistakes

- Believing `$b = $a` always makes an immediate deep copy.
- Believing objects are “passed by reference” and then adding `&` everywhere.
- Using references to optimize before measuring.
- Treating undefined, `null`, empty, zero, and false as one state.
- Reading superglobals deep inside domain code.
- Using globals or `$GLOBALS` as a service container.
- Assuming a static local is shared across servers or disappears after one queue message.
- Forgetting to remove a `foreach` reference after the loop.
- Letting user input select a variable name, class, method, include path, or SQL fragment.
- Keeping unbounded request data in a long-lived worker property.

## Senior Engineer Thinking

For every nontrivial variable, ask:

1. Who owns this value?
2. What is its exact lifetime?
3. Is it absent, null, empty, or valid with a zero-like value?
4. Does another variable share the same object or only the same logical value?
5. Can a mutation happen through an alias or hidden global?
6. Did this value cross a trust boundary, and what validation converted it?
7. Does it need to survive the process, or should it be durable shared state?

The mature design is often not “more immutable variables” or “never mutate.” It is choosing a clear owner and making mutation, sharing, and lifetime visible at the boundary where they matter.

## Exercises

1. Write a script that demonstrates array assignment, object assignment, `clone`, and reference assignment. Predict every output before running it.
2. Build a `ProfilePatch::fromInput()` value object that distinguishes an omitted field from an explicit `null` field. Test both cases.
3. Refactor a function that reads `$_POST` and a global database connection so that its core function receives typed arguments and its adapter performs input conversion.
4. Write a small worker loop that processes ten messages. Add an intentionally unbounded property cache, measure its growth, then replace it with message-local state or a bounded cache.
5. Find a use of `empty()` in application code and decide whether its broad rules match the domain. Replace it with a condition that states the intended rule.

## Review Questions

1. What does normal variable assignment mean for arrays and scalar values?
2. Why can two variables mutate the same object without being reference aliases?
3. When should `array_key_exists()` be used instead of `isset()`?
4. Why is `empty()` risky for identifiers and quantities?
5. What lifetime does a static local have in a long-running worker?
6. Why are superglobals input boundaries rather than domain objects?
7. Why does a PHP array used as a lock not coordinate multiple workers?
8. What can copy-on-write optimize, and when can mutation still increase memory?

## Summary

Variables are named runtime state with a scope, owner, and lifetime. PHP assignment is normally by value for scalar and array values, while object assignment copies a handle to the same object; explicit references create aliases and should be used only when the API needs them. Undefined, `null`, empty, zero, and false are distinct states that must not be collapsed accidentally. Read superglobals at boundaries, convert them into explicit values, keep process-local state bounded, and use databases or other shared systems for state that must cross process boundaries.

