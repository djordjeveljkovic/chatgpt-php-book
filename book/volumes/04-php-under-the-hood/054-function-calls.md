---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 54
title: Function Calls
slug: function-calls
status: complete
summary: ../../_ai/chapter-summaries/054-function-calls-summary.md
---

# Chapter 54 — Function Calls

## Why This Matters

`process($order)` looks like one operation in source code, but a call crosses several boundaries: argument expressions are evaluated, a callable is resolved, parameters are bound, a call frame is created, the callee executes, a result is produced, and temporaries are released. The same broad process applies to userland functions, methods, closures, and internal extension functions, although the execution path differs.

Knowing this helps explain parameter mutation, recursion, named arguments, callback overhead, stack traces, memory retention, and why a function that looks pure can still share an object or trigger I/O. It also gives an engineer a concrete place to investigate when a hot loop is dominated by dispatch, allocation, or repeated boundary crossings.

## Mental Model

```text
call expression
    ↓
evaluate arguments left-to-right
    ↓
resolve function/method/closure and scope
    ↓
bind positional, named, default, variadic, and reference parameters
    ↓
create execute frame
    ↓
run userland opcodes or enter internal function
    ↓
produce return zval or pending exception
    ↓
destroy temporaries and release the frame
```

This is a conceptual sequence. The VM and JIT may use specialized or inlined paths, but observable language rules and cleanup obligations remain.

## Core Concept: Arguments Are Values Before Binding

PHP evaluates argument expressions before calling the function, from left to right. The resulting values are then assigned to parameters, subject to type checks and reference requirements.

```php
<?php

declare(strict_types=1);

function mark(string $name): string
{
    echo $name, "\n";
    return $name;
}

function combine(string $first, string $second): string
{
    return $first . ':' . $second;
}

echo combine(mark('first'), mark('second'));
```

The output begins with `first`, then `second`. The callee does not start until both argument expressions have completed. This matters when arguments perform logging, mutate state, allocate data, or throw.

Parameter binding then applies defaults, named arguments, variadics, and declared types:

```php
<?php

function describe(string $id, bool $verbose = false): string
{
    return $verbose ? "full:$id" : "brief:$id";
}

echo describe(id: 'A-7', verbose: true);
```

Named arguments are part of the public parameter-name contract. Renaming a public parameter can therefore be a compatibility change even when its position is unchanged.

## What PHP Does With Different Value Kinds

By-value passing copies the zval-level value according to the type's rules. Arrays and strings can share their payload through copy-on-write; a write in the callee separates storage. Objects copy access to the same object instance. A by-reference parameter binds the callee to the caller's variable and can mutate it.

```php
<?php

function update(array $data, object $record, int &$counter): void
{
    $data['local'] = true; // COW separation; caller's array is unchanged.
    $record->state = 'updated'; // Shared object mutation.
    ++$counter; // Caller-visible variable mutation.
}
```

This is why the signature is a semantic contract, not just a memory hint. Chapters 51–53 cover references, COW, and object representation in detail.

## What Zend Does: Opcodes and `execute_data`

For a known userland call, compiled code commonly contains call setup and send/call opcodes such as `INIT_FCALL`, `SEND_*`, and `DO_FCALL` (exact opcode selection depends on the expression and compiler/runtime path). Methods and dynamic calls use related opcodes. The opcode names are useful for investigation, not a guarantee that every build executes an identical sequence after optimization.

The engine uses `zend_execute_data` as the call-frame structure. Current php-src initializes fields including the called `zend_function`, `$this`/called scope information, call flags, and argument count. A frame is placed on the Zend VM stack, with slots for arguments, locals, temporaries, and the function's execution bookkeeping.

```text
Zend VM stack
┌──────────────────────────────────────────┐
│ caller frame: locals / temporaries        │
├──────────────────────────────────────────┤
│ callee execute_data                       │
│   func → zend_function / op_array         │
│   This / called scope                     │
│   arguments                               │
│   locals and temporary zvals              │
├──────────────────────────────────────────┤
│ older frames                              │
└──────────────────────────────────────────┘
```

The frame is an implementation detail. Its layout, stack sizing, and fast paths can change. The important engineering model is that a call has state and lifetime; a retained closure, generator, Fiber, or stack trace can retain more than the source line suggests.

## Userland and Internal Functions

Userland functions execute compiled op_arrays through the Zend VM. An internal function is implemented by an extension and is entered through an engine call path with argument metadata and a return zval. The extension may perform I/O, allocate native memory, call back into PHP, or raise an exception.

At the C API level, `zend_call_function()` accepts call information and a function cache; helpers such as `zend_call_known_function()` invoke a known function or method. These APIs are for extension code and are version-sensitive. They show that callable resolution and invocation are explicit runtime activities, not a magical property of a PHP identifier.

## Return Values and Cleanup

The callee writes or returns a zval result. The caller consumes it, assigns it, returns it, or discards it. Reference-counted values require the correct add-reference and destruction operations as values move between slots. If an exception is pending, the normal return path is abandoned and the engine begins exception handling.

```php
<?php

function makePayload(): array
{
    return ['ready' => true];
}

$payload = makePayload();
```

The language says `$payload` receives the returned array. The engine can transfer or reuse temporary storage in some situations, but application code should not depend on a particular move optimization. Optimize around measured allocations and lifetimes, not around guessed pointer transfers.

## Recursion and Re-entrancy

Each active call needs a frame. Recursion therefore consumes stack/frame space and can fail at a depth or memory limit long before the algorithm's data structure is large.

```php
<?php

function depth(int $n): int
{
    return $n === 0 ? 0 : 1 + depth($n - 1);
}
```

Extension functions and callbacks can re-enter userland. A supposedly simple internal operation may invoke a user-defined comparator, error handler, magic method, or destructor. When writing extensions or reviewing callback-heavy code, account for re-entrancy: global state may change, exceptions may be raised, and user code can call back into the same service.

## Bad Example: Work Hidden in Arguments

```php
saveInvoice(
    invoice: $repository->find($request->id),
    total: $pricing->recalculate($request->id),
    audit: $audit->record('save started'),
);
```

The call site hides three operations with independent failure and ordering behavior. An exception in the second expression prevents the function call; the audit call may already have happened. The code may be valid, but its transaction and observability story is difficult to read.

## Better Example: Name the Boundary

```php
<?php

$invoice = $repository->find($request->id);
$total = $pricing->recalculate($invoice);
$audit->record('save started', ['invoice_id' => $invoice->id]);

$service->saveInvoice(
    invoice: $invoice,
    total: $total,
);
```

This does not reduce the number of calls. It makes evaluation order, failure points, and measured timings visible. If the operations need one transaction, define that transaction around the correct unit rather than assuming the function call creates one.

## Production Example: Callback Cost

```php
<?php

/** @param list<int> $values */
function sumPositive(array $values): int
{
    $sum = 0;
    foreach ($values as $value) {
        if ($value > 0) {
            $sum += $value;
        }
    }
    return $sum;
}
```

For a hot loop, this can be easier to profile than repeated higher-order callbacks. That is not a universal rule: built-ins can be implemented efficiently, and clarity may dominate. Measure with realistic data, including the cost of allocations and callback dispatch, before replacing an expressive operation.

## Performance

An individual call includes resolution, argument preparation, frame management, parameter checks, execution, and cleanup. In most applications, database and network latency dominate. In a tight CPU loop, millions of tiny calls or dynamic callbacks can become material.

```text
Cheap enough?        → keep the clear function boundary.
Hot and measurable?  → inspect profile data; reduce repeated work or batch.
Dynamic and hot?     → cache resolution or use a direct path if justified.
Recursive and deep?  → prove a safe depth or use an iterative algorithm.
```

OPcache and JIT may optimize calls in a particular deployment, but never assume that a microbenchmark on the CLI predicts FPM, a worker, or a JIT-enabled production build. Compare the same PHP version, configuration, input, and warm-up state.

## Security and Operations

Dynamic calls turn data into control flow. Validate callable names and never construct a callable from untrusted input without an allowlist. A function call can also cross into an extension that reads files, opens sockets, or invokes a user callback; apply the relevant capability and timeout controls.

For production traces, record function boundaries selectively. Full stack traces and argument dumps can be expensive and can disclose credentials or personal data. Prefer operation names, IDs, durations, and exception classes with redacted context.

## Testing

Test evaluation order when arguments have side effects:

```php
<?php

$events = [];

function event(array &$events, string $name): string
{
    $events[] = $name;
    return $name;
}

function pair(string $first, string $second): array
{
    return [$first, $second];
}

pair(event($events, 'first'), event($events, 'second'));
assert($events === ['first', 'second']);
```

Test by-reference parameters for caller mutation, object parameters for shared identity, named-argument compatibility, exception paths before callee entry, and cleanup after a failed call. Performance tests should distinguish userland dispatch, internal calls, database time, and serialization/network time.

## Common Mistakes

- Treating a function call as a single indivisible operation.
- Assuming by-value arrays are eagerly deep-copied.
- Assuming objects are copied because the parameter lacks `&`.
- Renaming public parameters without considering named callers.
- Measuring a CLI call and generalizing to every SAPIs/configuration.
- Forgetting that callbacks, magic methods, error handlers, and destructors can re-enter userland.

## Senior Engineer Thinking

For a suspicious call, draw the boundary:

```text
What is evaluated? What can throw? What is shared?
What frame/temporary/closure is retained?
What work is CPU, database, network, or serialization?
Can it be batched without changing failure semantics?
```

The goal is not to eliminate calls. It is to make call costs, ownership, failure, and observability proportional to the system's constraints.

## Exercises

1. Use `php -d opcache.opt_debug_level=...` or a suitable opcode inspection tool in a disposable environment to compare a direct and dynamic function call. Treat output as version-specific.
2. Write a recursive tree walk and an iterative equivalent. Measure memory and failure behavior at increasing depth.
3. Create a function with side-effecting arguments and write a test that documents evaluation order.
4. Profile a callback-heavy transformation and compare it with a direct loop. Report where the time actually went.

## Review Questions

1. In what order are PHP function arguments evaluated?
2. What conceptual responsibilities does an `execute_data` call frame hold?
3. How do arrays, objects, and references differ when bound to parameters?
4. Why can a function call fail before the callee starts?
5. When is reducing function-call overhead a justified optimization?

## Summary

A PHP function call evaluates arguments, resolves a callable, binds parameters, creates execution state, runs userland or internal code, returns a zval or exception, and cleans up. The Zend VM represents active calls with `zend_execute_data` frames and uses call opcodes and extension APIs to enter functions. Arrays and strings can use COW across by-value boundaries, objects share identity, and references mutate caller variables. Use profiles to find hot boundaries, keep side effects visible, and treat dynamic calls, recursion, callbacks, and retained frames as explicit costs.

## Official References

- [PHP Manual: Function parameters and arguments](https://www.php.net/manual/en/functions.arguments.php)
- [PHP Manual: Named arguments](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments)
- [php-src: VM call-frame helpers](https://github.com/php/php-src/blob/master/Zend/zend_execute.h)
- [php-src: function-call API](https://github.com/php/php-src/blob/master/Zend/zend_API.h)
- [php-src: call-related opcodes](https://github.com/php/php-src/blob/master/Zend/zend_vm_opcodes.h)
