---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 16
title: Scope
slug: scope
status: complete
summary: ../../_ai/chapter-summaries/016-scope-summary.md
---

# Chapter 16 — Scope

## Why This Matters

Scope answers a deceptively practical question: which code can see, read, and change a variable? The answer determines ownership, lifetime, testability, and the chance that one request or job contaminates another.

PHP has global scope and function scope rather than the block scope found in some languages. It also has closure captures, class scope, namespace-qualified names, static locals, static properties, and included-file behavior. These mechanisms are powerful, but they can make state look more local or more shared than it really is.

Scope is therefore an operational concern:

```text
where a name is visible
        ↓
who can mutate its value
        ↓
how long the value remains reachable
        ↓
whether another request or worker can observe it
```

A variable in one PHP-FPM worker is not shared with another worker. A static local in a long-running worker can nevertheless survive across many messages. A global can be imported into a function by reference and silently change process-level state. Clear scope makes those boundaries reviewable.

## Mental Model

Separate four ideas that are often collapsed into the word “scope”:

1. **Name visibility:** where a variable name can be resolved.
2. **Mutation authority:** which code can change the value.
3. **Lifetime:** how long the value remains reachable.
4. **Process sharing:** whether another execution context can observe it.

For example, a local variable has narrow name visibility and usually a short lifetime, but an object it references may contain a connection to a shared database. A static property has class-level access rules but lives as process state. A database row is not a PHP variable, even though a variable may hold its hydrated representation.

## Core Concept

### Global and function scope are different

Variables declared outside a function are in the current file’s global scope. A variable declared inside a named function is local to that function invocation:

```php
$defaultSurface = 'clay';

function surfaceLabel(): string
{
    // $defaultSurface is not automatically visible here.
    return 'Unknown';
}
```

Calling a function creates a new local environment. Assigning `$defaultSurface` inside the function would create a local variable, not change the outer one. This is one reason explicit parameters are clearer:

```php
function surfaceLabel(string $surface): string
{
    return ucfirst($surface);
}
```

PHP does not give ordinary variables block scope:

```php
if ($enabled) {
    $message = 'Enabled';
}

echo $message; // The name can still be visible after the block.
```

Do not rely on this as a reason to declare values anywhere. Initialize variables along clear paths and keep temporary names in small functions so their lifetime is apparent to humans and tools.

### `global` imports a reference, not a copy

The `global` keyword makes a local name refer to a global variable:

```php
$requestCount = 0;

function recordRequest(): void
{
    global $requestCount;
    $requestCount++;
}
```

The function can mutate the outer variable. This is usually a hidden dependency and makes tests order-dependent. Pass state explicitly or put owned mutable state behind an object whose lifecycle is constructed at the composition boundary.

`$GLOBALS` is another way to access global variables and is a superglobal. It is not a safer service container. The fact that a function can reach a value does not mean it should own that dependency.

### Static locals persist between calls

A local declared with `static` retains its value between calls in the same process:

```php
function nextSequence(): int
{
    static $sequence = 0;

    return ++$sequence;
}
```

This can be useful for a deliberately process-local memoization or an implementation detail. It is not a durable sequence, a cluster-wide counter, or a request-safe lock. In a normal short-lived request, its persistence may be invisible. In a queue worker, it can span thousands of messages.

Do not put request-specific data in static locals unless the worker lifecycle and reset policy are explicit. If the state must survive a process restart or coordinate workers, use a database, cache, or other shared system.

### Closures capture outer variables deliberately

An anonymous function has its own function scope. It does not automatically see ordinary variables from the defining scope unless it captures them:

```php
$taxRate = 0.2;

$withTax = function (int $cents) use ($taxRate): int {
    return (int) round($cents * (1 + $taxRate));
};
```

The value is captured by value. A reference capture observes later mutation:

```php
$calls = 0;

$record = function () use (&$calls): void {
    $calls++;
};

$record();
echo $calls; // 1
```

Use reference capture only when shared mutable state is the intended contract. Otherwise pass a value as an argument or return the updated state. Arrow functions capture used variables automatically by value:

```php
$taxRate = 0.2;
$withTax = fn (int $cents): int => (int) round($cents * (1 + $taxRate));
```

An arrow function cannot use a `use` clause or capture by reference. It can still access `$this` when created in an object context, and a closure can retain an object graph longer than expected.

### Class scope controls members, not ordinary local variables

Class methods have access to the class’s private and protected members according to visibility rules. A property belongs to an object instance or to the class for a static property; it is not the same as a local variable:

```php
final class RequestCounter
{
    private static int $count = 0;

    public function record(): int
    {
        return ++self::$count;
    }
}
```

The static property is shared by instances of the relevant class in the process, subject to inheritance semantics. It is not shared among PHP processes and should not represent durable business state. An instance property is usually easier to reset and test; a dependency such as a counter store is better when the count has application meaning.

### Namespace is name resolution, not variable isolation

A namespace qualifies functions, classes, interfaces, traits, enums, and constants. It does not create a separate process or database and should not be treated as a variable-isolation mechanism:

```php
namespace App\Reservations;

$value = 'local to this file execution';
```

Use imported names and fully qualified names to control symbol resolution. A namespaced function can still use globals, superglobals, static state, and included-file variables if the source asks it to. Namespace boundaries help organize symbols; dependency boundaries still need design.

### Included files inherit the scope of the include site

An included file executes in the scope where `include` or `require` is called:

```php
$config = ['timezone' => 'UTC'];
require __DIR__ . '/bootstrap-fragment.php';
```

The fragment can see `$config` if it is included at global scope. If the same fragment is included inside a function, it sees that function’s local scope instead. This is one reason library files should expose functions or return values rather than depend on variables that happen to exist at the include site.

`require` and `include` are execution, not textual preprocessing in the simplistic sense. The included code runs when the statement is reached, and its return value can be captured:

```php
$config = require __DIR__ . '/config.php';
```

Keep config files side-effect free and return explicit data. Never build an include path from untrusted input.

## How It Works

The Zend Engine resolves variable names against an execution context. A function call creates a frame with its local variables. A closure carries captured variables and scope metadata. A method call additionally has an object context (`$this`) and class visibility context. Static locals and static properties are stored so later calls can reach them within the same process.

These are runtime structures, not independent shared-memory segments. PHP-FPM workers normally have separate processes. OPcache can share compiled code and some memory between processes, but ordinary userland variable state is not a cross-worker variable. Volume IV develops zvals, references, and call frames; Volume V returns to worker lifetime and process memory.

## What PHP Does

The main scope mechanisms are:

| Mechanism | Visibility | Lifetime concern |
| --- | --- | --- |
| Local variable | Current function invocation | Usually until unreachable |
| Global variable | Top-level file execution and explicit imports | Process/request state |
| Static local | Function body across calls | Entire process |
| Closure capture | Closure body | Closure lifetime |
| Instance property | Methods with object access | Object lifetime |
| Static property | Class methods/access allowed by visibility | Class/process lifetime |
| Superglobal | Available across scopes | Request/environment dependent |
| Namespace | Symbol resolution | Does not isolate variable state |
| Included-file variables | Include-site scope | Duration of execution context |

Superglobals such as `$_GET`, `$_POST`, `$_SERVER`, `$_SESSION`, and `$argv` are available across function scopes, but availability and contents depend on the entry point. Read them in an adapter and pass normalized values onward. “Global” availability is not a reason for domain code to depend on transport state.

## What Zend Does

Variable access is represented by runtime value slots and scope-aware lookups. Explicit references can bind names to the same variable container, while ordinary array assignment uses copy-on-write and object assignment copies a handle. A closure’s captured variable can therefore keep a value reachable, and a static value can outlive every local call that first created it.

The conceptual model is enough for application design:

```text
local variable       → call frame
static local         → function-associated process state
instance property    → object state
static property      → class-associated process state
global               → top-level process/request state
database row         → durable shared state outside PHP
```

Do not infer cross-process sharing from the fact that two workers executed the same source code.

## Minimal Example

```php
<?php

declare(strict_types=1);

function greeting(string $name): string
{
    $trimmed = trim($name);

    return "Hello, {$trimmed}";
}

echo greeting('Ada'), PHP_EOL;
```

`$trimmed` is local to one call. The caller owns `$name` and receives a new string. There is no hidden state to reset between tests.

## Practical Example

Replace an implicit global dependency with an explicit object:

```php
final class ReservationPolicy
{
    public function __construct(private readonly Clock $clock)
    {
    }

    public function canStart(ReservationCommand $command): bool
    {
        return $command->startsAt >= $this->clock->now();
    }
}
```

The clock is constructed at the composition boundary and can be replaced by a fixed test clock. The policy object’s lifetime is explicit: create one per request or inject a deliberately shared instance into a worker. Neither choice should be accidental.

## Production Example

A worker should reset message-local state by keeping it local to the handler:

```php
final class ImportWorker
{
    public function __construct(private readonly Importer $importer)
    {
    }

    public function run(MessageSource $messages): void
    {
        foreach ($messages as $message) {
            $report = $this->handle($message);

            // Acknowledge only after the durable result is recorded.
            $messages->acknowledge($message->id, $report->status);
        }
    }

    private function handle(Message $message): ImportReport
    {
        $rows = $message->decodeRows();

        return $this->importer->import($rows);
    }
}
```

`$rows` and `$report` are per-message locals. The worker object retains only its importer dependency. If the importer has caches, their bounds and reset policy must be explicit. A process supervisor can recycle workers, but recycling is a safety net rather than a substitute for ownership.

## Bad Example

```php
$db = connectDatabase();
$currentUser = loadUser();

function reserve(): void
{
    global $db, $currentUser;

    static $seen = [];
    $courtId = $_POST['court_id'];

    if (!isset($seen[$courtId])) {
        $seen[$courtId] = true;
        $db->exec("INSERT INTO reservations (court_id) VALUES ($courtId)");
    }
}
```

This function mixes global references, process-persistent state, request input, SQL construction, and a false assumption that `$seen` prevents other workers from inserting. It can also leak behavior from one queue message to another if reused in a long-running process. The scope is broad while the contract is invisible.

## Better Example

```php
function reserve(
    ReservationCommand $command,
    ReservationRepository $repository,
    Authorization $authorization,
    UserId $actor,
): ReservationId {
    $authorization->assertCanReserve($actor, $command->courtId);

    return $repository->insertIfAvailable($command);
}
```

Input conversion occurs before this function, dependencies are visible, and the repository can define the transaction and conflict behavior. The durable repository operation—not a static local—must protect the shared invariant.

## Edge Cases

- PHP has function and global variable scope, not ordinary block scope. A variable created in an `if` block may remain available afterward.
- A function does not automatically see variables from global scope. Use a parameter rather than `global` for ordinary dependencies.
- `global $name` imports a reference to the global variable; mutation affects the global state.
- `$GLOBALS` is a superglobal and exposes global state from any scope. It increases coupling rather than solving it.
- A static local persists across calls in one process. It is reset when the process exits, not necessarily after one web request in a worker model.
- Static properties are class/process state, not shared database state. Inheritance and `self::`/`static::` can affect which class’s property is accessed.
- Closures capture by value with `use ($value)` and by reference with `use (&$value)`. The latter observes later mutations.
- Arrow functions capture used outer variables by value and cannot capture by reference.
- A closure created in an object method may retain `$this` and its object graph.
- Included code inherits the scope of the include site. A file included inside a function can see that function’s locals.
- Namespaces qualify symbols but do not isolate ordinary variables or process state.
- Superglobals vary between CLI, web, and worker entry points; never assume `$_POST` exists or has a particular shape.
- A `foreach` reference can keep the loop variable aliased after the loop. Avoid it or call `unset()` promptly.

## Performance

Local variables are usually the easiest state for PHP to reclaim when a call ends. Large arrays retained by static locals, static properties, globals, object properties, or closure captures remain reachable and can raise peak or steady-state worker memory.

Scope itself is rarely the bottleneck. Hidden scope makes performance harder to measure: a global cache may turn an O(1) lookup into unbounded memory, and a closure capture may retain an entire service graph. Choose cache scope based on invalidation, memory budget, process model, and consistency requirements. Use a shared cache when data must cross workers, and bound every process-local cache.

## Security

Broad scope increases the number of paths that can reach sensitive state. Avoid globals containing credentials, current-user data, raw request payloads, or database handles. Do not let user input select a variable name, callable, class, method, or include path. Read and validate superglobals at the boundary, then pass an allow-listed value into domain code.

Scope is not authorization. A private property can still hold data that the current actor must not see, and a local `$isAdmin` can be incorrect if it was derived from an untrusted request field. Perform authorization against trusted identity and durable resource state before the side effect.

## Database Interaction

A PHP scope cannot replace database scope. A local `$hasConflict` result is valid only for the read that produced it. Another request can change the row immediately afterward. Use a transaction, appropriate isolation/locking, unique or exclusion constraints where supported, and a clear retry policy.

Similarly, a static property used as a cache can become stale independently in every worker. If stale data is acceptable, document the consistency window. If it is not, use invalidation or a shared source of truth. The placement of a variable does not define durability or consistency.

## Concurrency

Ordinary PHP variables are isolated per process, which is useful for request-local state. PHP-FPM can run several processes concurrently, and queue workers can execute the same function at the same time. Globals and statics do not coordinate those processes.

Within one process, a long-running worker makes static and captured state visible across messages. Reset message-specific state explicitly, bound caches, and make handlers idempotent. Across processes, coordinate through a database, cache, filesystem protocol, or message broker with the atomicity and failure semantics required by the invariant.

## Testing

Test scope through observable ownership and lifetime:

- call a function twice and verify whether a static local is intentionally retained or must reset;
- run a worker handler for multiple messages and detect state leakage between them;
- test closure captures by value and reference when callbacks are part of the contract;
- test that a closure does not retain an unintended large object graph in a long-running process;
- test instance and static properties with fresh objects and process-isolation when needed;
- test included configuration through its return value rather than accidental caller variables;
- replace global dependencies with explicit fakes and test the adapter separately;
- test database races with integration/concurrency tests, not only process-local branches.

Static analysis can detect undefined variables, invalid captures, and some global usage. A unit test that runs in a fresh process may miss the worker-only lifetime bug; include a multi-message worker fixture for that case.

## Common Mistakes

- Assuming variables in an `if` or `foreach` block disappear afterward.
- Using `global` or `$GLOBALS` as dependency injection.
- Treating a static local as a durable or cross-worker counter.
- Capturing mutable state by reference without documenting ownership.
- Letting closures retain request or service graphs in workers.
- Confusing namespace organization with variable or process isolation.
- Including files that depend on arbitrary include-site variables.
- Reading superglobals deep inside business logic.
- Using process-local state as a database lock or uniqueness check.
- Testing only short-lived requests and missing worker state leakage.

## Senior Engineer Thinking

For every nontrivial value, ask:

1. Where is its name visible, and who can mutate it?
2. When does it become unreachable or reset?
3. Can a closure, static, property, or global retain it unexpectedly?
4. Is the state request-local, worker-local, process-shared, or durable?
5. Does another worker need to observe or coordinate with it?
6. Can the dependency be passed explicitly and tested with a fake?
7. What happens after a crash, restart, retry, or deployment?

Scope design is ownership design. The best default is narrow visibility, explicit dependencies, bounded lifetimes, and durable shared state for invariants that must survive or coordinate beyond one process.

## Exercises

1. Write a script demonstrating global scope, function scope, block visibility, `global`, and `$GLOBALS`. Refactor the global dependency into a parameter.
2. Build a static-local memoizer, then run it in a simulated multi-message worker. Decide whether its lifetime and invalidation policy are acceptable.
3. Create closures that capture a threshold by value and by reference. Mutate the outer variable and explain each result.
4. Move a configuration include from accidental variable sharing to a file that returns a typed array or value object. Test both CLI and web-style callers.
5. Find a process-local reservation lock and replace it with a database invariant or transaction. Explain why the original scope could not coordinate workers.

## Review Questions

1. What are PHP’s primary variable scopes?
2. Why is a variable inside an `if` block potentially visible afterward?
3. What does `global` do, and why can it make tests difficult?
4. How long can a static local live?
5. How do closure value and reference captures differ?
6. What is special about arrow-function captures?
7. Why is a namespace not a variable-isolation boundary?
8. What scope does an included file inherit?
9. Why can a static property not be a cross-worker lock?
10. How should a long-running worker handle message-local state?

## Summary

PHP scope governs name visibility, mutation paths, lifetime, and process-local state. Functions do not automatically see globals; `global` imports a mutable reference; static locals and properties persist within a process; closures capture by value or reference; arrow functions capture by value; included files inherit include-site scope; and namespaces organize symbols without isolating variables. Prefer explicit parameters and dependencies, keep worker state bounded, read superglobals at adapters, and use durable shared systems for invariants that cross process boundaries.

## Official References

- [PHP Manual: Variable scope](https://www.php.net/manual/en/language.variables.scope.php)
- [PHP Manual: Anonymous functions](https://www.php.net/manual/en/functions.anonymous.php)
- [PHP Manual: Arrow functions](https://www.php.net/manual/en/functions.arrow.php)
- [PHP Manual: `include`](https://www.php.net/manual/en/function.include.php)
