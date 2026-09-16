---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 46
title: Zend VM
slug: zend-vm
status: complete
summary: ../../_ai/chapter-summaries/046-zend-vm-summary.md
---

# Chapter 46 — Zend VM

## Why This Matters

Chapter 45 showed that compilation produces opcodes. The next question is practical: who executes those opcodes, and what state must be preserved while they run? The Zend virtual machine (VM) is the part of the Zend Engine that turns an op array into observable PHP behavior.

This explains several otherwise surprising observations:

- a function call is a transfer to another execution context, not merely a text substitution;
- a local variable, a temporary result, and a function return value need different storage and cleanup rules;
- an exception can abandon the ordinary next-opcode path and unwind several calls;
- an internal function crosses from VM-managed userland execution into extension code;
- the same source can have different VM-level details in different PHP builds.

The VM is an implementation of PHP’s language rules. It is not the language specification, and it is not a CPU emulator that executes native machine instructions one by one.

## Mental Model

```text
compiled zend_op_array
        │
        ▼
execution context (current function, opline, locals, return slot)
        │
        ▼
fetch operands → run one VM handler → write result/update control flow
        │                                      │
        └────────────── next opline ◄──────────┘
                         │
             call / return / throw / suspend
```

Conceptually, the executor repeats this cycle:

```text
while there is an instruction to run:
    read the current opcode and operands
    perform the operation using zvals and runtime handlers
    release or preserve temporaries as required
    choose the next instruction or transfer control
```

The loop is deliberately conceptual. Current php-src generates executor code and may use specialized dispatch strategies; it is not a promise that a literal `while` loop appears in the binary.

## Core Concept

An execution context represents a currently running function-like unit. In current Zend source, `zend_execute_data` contains fields used to identify the function, current opcode, caller context, return value, and other call-state information. The exact layout is private and version-sensitive. It is useful to picture a frame like this:

```text
execute_data
├── function / method metadata
├── current opline
├── previous execution context
├── return-value destination
├── `$this` / called scope when applicable
└── local variables and temporary VM slots
```

A frame is not a PHP object visible to application code. It is runtime bookkeeping. Debuggers and stack traces expose selected, language-level views of it; they do not expose a stable C structure.

The VM works with zvals, which are the engine’s tagged runtime values. A zval may contain an integer or boolean directly, or a pointer to a refcounted string, array, object, or other runtime payload. Chapter 47 develops that representation. The VM therefore coordinates both instruction dispatch and memory ownership.

## How It Works

Consider:

```php
<?php

declare(strict_types=1);

function lineTotal(int $quantity, int $unitPrice): int
{
    return $quantity * $unitPrice;
}

$total = lineTotal(3, 120);
echo $total, PHP_EOL;
```

The conceptual execution sequence is:

```text
top-level frame:
    prepare call to lineTotal
    place 3 and 120 in argument positions
    call lineTotal

lineTotal frame:
    read quantity and unitPrice
    multiply the two zvals
    place result in return slot
    return to caller

top-level frame:
    receive returned zval in $total
    convert/output it through echo semantics
```

The VM does not copy the function’s source text into the caller. The callee has its own compiled representation and execution state. Argument passing, type checks, reference semantics, cleanup, and return-value transfer are all part of the call machinery.

### Dispatch and handlers

An opcode identifies an operation such as a jump, assignment, call, return, array access, or arithmetic operation. A VM handler implements the operation. In a simplified representation:

```text
current = frame->opline

switch (current->opcode) {
    case ADD:
        result = add_with_php_semantics(op1, op2);
        frame->opline++;
        break;
    case JMP:
        frame->opline = current->target;
        break;
    case RETURN:
        leave_frame(frame);
        break;
}
```

Real generated handlers do more: they select operand modes, invoke object or extension handlers, manage reference counts, record exceptions, and arrange fast paths. Depending on the build, dispatch can use a switch, computed-goto-like labels, or generated specialized code. This is implementation-specific and should be verified against the exact php-src version being studied.

### Calls, returns, and internal functions

Userland calls enter another op array. A call to an internal function instead enters a C implementation registered by an extension or by the core. The language-level result can be the same:

```php
$length = strlen($name);      // internal function boundary
$total = lineTotal(3, 120);   // userland function boundary
```

The performance and failure paths differ. An internal function may operate directly on engine values and call a library. A userland function requires a new userland execution context and its own opcode dispatch. Neither category is automatically “fast”; argument conversion, allocation, hashing, I/O, and downstream work may dominate.

### Control flow and exceptions

For a conditional, the VM changes the next opline:

```text
evaluate condition
    ├── false → jump to else/end
    └── true  → execute then block → jump over else block
```

For an exception, ordinary sequential dispatch stops. The engine searches for a matching handler, performs required cleanup, and resumes at a catch or finally region. If no handler is found in the current frame, control continues through callers. This is why a stack trace is a path through execution contexts rather than a list of source lines that independently failed.

Generators and fibers complicate the simple “frame ends at return” picture. A generator can suspend with its execution state and resume later; a fiber can cooperatively suspend and transfer control to its caller. Their public semantics are defined by PHP, while the precise saved structures and VM transitions are implementation details.

## What PHP Does

PHP defines what a program can observe: evaluation rules, return values, output, errors, exceptions, side effects, and lifecycle behavior. PHP does not promise particular opcode numbers, dispatch strategy, frame offsets, or handler names.

For example, these two implementations can have different VM work while preserving the same result for ordinary inputs:

```php
$total = $quantity * $price;

$total = intdiv($quantity * $price, 1);
```

They are not necessarily semantically identical for all inputs or errors, and an optimizer may treat them differently. The right test is the language-visible behavior under the domain’s input and failure matrix, not a preferred-looking instruction trace.

## What Zend Does

The executor and VM definitions live primarily in the `Zend/` portion of php-src. In a PHP 8.4 source tree, [Zend/zend_execute.c](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_execute.c) contains execution support, while [Zend/zend_vm_def.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_vm_def.h) contains VM handler definitions used to generate executor code. [Zend/zend_vm_execute.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_vm_execute.h) is generated/build-specific rather than a stable extension API.

The names and layout of `zend_execute_data`, VM stack helpers, handlers, and dispatch macros can change between releases, build options, and optimization work. When investigating an incident, pin the source tag to the PHP binary and record whether OPcache/JIT is enabled. A diagram from this chapter is a model for reasoning, not a supported ABI.

## Minimal Example

This program contains a branch, a call, and a return:

```php
<?php

declare(strict_types=1);

function classify(int $value): string
{
    if ($value < 0) {
        return 'negative';
    }

    return 'non-negative';
}

echo classify(-2), PHP_EOL;
```

The VM-level shape is approximately:

```text
classify frame:
    receive value
    compare value with 0
    jump when comparison is false
    return literal string "negative"
    return literal string "non-negative"

top-level frame:
    call classify
    echo returned value
```

“Approximately” matters: literal representation, comparison specialization, and return cleanup are version/build details.

## Practical Example: Finding the Boundary That Is Slow

Suppose a report endpoint is slow. Do not begin by counting opcodes. First split the path:

```text
request
  → controller/userland calls
  → array/object manipulation
  → internal extension call
  → database/network wait
  → serialization/output
```

Use a profiler or timing instrumentation to identify the dominant segment. VM knowledge becomes useful when the evidence points to repeated userland calls, conversions, array access, or allocation. If the process spends 180 ms waiting for a database, shaving a few dispatches from a 2 ms PHP loop is irrelevant.

For a controlled experiment, keep the boundary visible:

```php
<?php

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

Benchmark this with representative values, then compare a different algorithm or data structure if the input is large. A profiler can tell you whether the time is in VM execution, HashTable access, zval separation, a function call, or an extension.

## Bad Example

```text
The VM uses a jump for this if-statement, so replacing it with a clever
boolean expression is guaranteed to be faster.
```

This confuses an implementation observation with an optimization conclusion. The rewrite may change evaluation order, readability, error behavior, allocations, or branch predictability. It may also be lost or transformed by a later compiler/optimizer pass.

## Better Example

State a hypothesis and measure the full cost:

```text
symptom: a 5-million-row in-memory loop is CPU-bound
    → verify with a profiler
    → inspect VM calls, array access, conversions, and allocations
    → compare an indexed lookup or database-side operation
    → benchmark warm and cold runs with peak memory
    → retain the clearer implementation unless the measured gain matters
```

The best optimization may be moving filtering into SQL, choosing a packed list instead of a keyed map, or avoiding repeated object construction—not rearranging source to chase a particular opcode sequence.

## Edge Cases

- A call can throw before the callee’s body runs, during argument evaluation, inside the callee, or during return-value handling.
- A VM handler can invoke overloaded object behavior or an extension, so one opcode can trigger substantial user code or I/O.
- `finally` code runs on normal return and many non-local control transfers; cleanup is part of the semantics.
- A generator or fiber preserves execution state across a suspension boundary, so a single request may revisit an op array later.
- Debugging/profiling instrumentation can change timings and sometimes execution paths.
- OPcache and JIT can alter the warm execution path. JIT is an optional implementation optimization, not a requirement for PHP semantics.
- A fatal error, memory exhaustion, or process termination may prevent ordinary userland cleanup and logging.

## Performance

VM dispatch has a cost, but it is only one term in a PHP request’s budget:

```text
CPU ≈ dispatch + operand/value work + allocation + extension work + I/O wait
```

Measure wall time and CPU time separately. Record PHP version, SAPI, OPcache/JIT configuration, input size, process warm-up, and external service behavior. Compare algorithms before micro-optimizing instruction-shaped code. Peak memory matters because a faster implementation that doubles worker memory can reduce pool capacity.

## Security

VM details do not make untrusted PHP safe. Opcode inspection is not a sandbox, and hiding a dangerous call behind a helper does not remove its effect. If an application evaluates user-controlled source, the security boundary includes the PHP process, loaded extensions, filesystem, environment, and operating-system permissions. Prefer not to evaluate untrusted PHP at all; use a constrained data format and explicit interpreter when the product requires user-defined rules.

Diagnostic dumps can expose source paths, literals, class names, and secrets embedded in code. Restrict them to controlled development or incident environments and redact output before sharing.

## Testing

Test the language-visible behavior at normal application boundaries:

1. Test return values, output, exceptions, and side effects.
2. Test nested calls and `finally` behavior on success and failure.
3. Test generators/fibers separately when suspension and resumption are part of the design.
4. Use `php -l` for syntax fixtures and a profiler/benchmark for performance hypotheses.
5. Snapshot opcode or VM output only for version-pinned engine/tooling tests; treat changes as review signals, not automatically as bugs.

A useful fixture records the PHP binary and configuration:

```bash
php -v
php --ini
php -i | grep -E 'opcache.enable|opcache.jit'
php -l tests/fixtures/vm-path.php
```

## Common Mistakes

- Treating Zend VM instructions as native CPU instructions.
- Assuming an opcode count predicts request latency.
- Treating `execute_data` or generated executor headers as a stable extension ABI.
- Forgetting internal-function, database, and network boundaries.
- Assuming a function call is free because OPcache is enabled.
- Assuming a thrown exception is just a returned error value.
- Using VM internals to justify a rewrite without a representative measurement.

## Senior Engineer Thinking

The VM is a useful layer in a causal explanation:

```text
source choice
    → compiled control/data flow
    → VM operations and value handling
    → allocations/calls/extension work
    → observed latency and memory
```

Stop at the layer that explains the symptom. If an indexed database query fixes the problem, explain that result in terms of cardinality and I/O rather than claiming a magical VM improvement. If a hot pure-PHP loop is the bottleneck, VM, zvals, and HashTables can explain why a data-structure change is effective.

## Exercises

1. Draw the conceptual execution contexts for a top-level script calling a userland function that calls `strlen()`.
2. Sketch the VM control flow for `try/catch/finally` and identify where ordinary sequential dispatch can be interrupted.
3. Write a small generator and annotate which values and control state must survive `yield`.
4. Benchmark a loop that performs repeated associative-array lookups. Record PHP version, OPcache state, time, and peak memory; then compare an indexed representation.
5. Inspect a pinned php-src version and locate the executor entry point, a VM handler definition, and the execution-context structure. Note which details are not safe to depend on from an extension.

## Review Questions

1. What does the Zend VM execute, and how is that different from native machine code?
2. What information does an execution context need to resume a function?
3. Why can an internal function and a userland function have different costs even when their PHP-level calls look similar?
4. Why does an exception require more than choosing the next sequential opcode?
5. Which VM facts are useful for a performance investigation, and which are too unstable for application design?

## Summary

The Zend VM executes compiled op arrays using execution contexts, operand values, handlers, control-flow transitions, call/return machinery, and cleanup paths. Its dispatch strategy and private structures are implementation-specific and version-sensitive. Use VM knowledge to explain measured behavior—especially calls, allocations, value handling, and exceptions—while testing the stable PHP semantics at application boundaries.

## References

- [php-src PHP-8.4: `Zend/zend_execute.c`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_execute.c)
- [php-src PHP-8.4: `Zend/zend_vm_def.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_vm_def.h)
- [php-src PHP-8.4: `Zend/zend_vm_execute.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_vm_execute.h)
- [php-src PHP-8.4: `Zend/zend_types.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h)
- [PHP Manual: language execution and errors](https://www.php.net/manual/en/language.errors.php)
- [PHP Manual: generators](https://www.php.net/manual/en/language.generators.php)
- [PHP Manual: Fibers](https://www.php.net/manual/en/language.fibers.php)
