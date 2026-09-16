---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 45
title: Opcodes
slug: opcodes
status: complete
summary: ../../_ai/chapter-summaries/045-opcodes-summary.md
---

# Chapter 45 — Opcodes

## Why This Matters

An opcode is a VM-level instruction. Opcodes provide the most concrete view so far of how PHP expresses work after parsing and compilation. They explain why an innocent-looking statement can involve fetches, temporaries, calls, jumps, cleanup, and exception bookkeeping.

Opcode knowledge is useful for profiling and engine work, but it has a trap: opcode names, numbering, operands, and optimizer output are implementation-specific. Use them to form and test a performance hypothesis, not to write application logic that depends on an instruction sequence remaining unchanged.

## Mental Model

```text
zend_op {
    opcode        → operation kind
    op1, op2      → inputs or control-flow data
    result        → destination
    operand types  → constant, compiled variable, temporary, or variable
    line/metadata → diagnostics, calls, caches, and cleanup
}
```

A compiled function is an ordered op array. The VM maintains execution state while it dispatches one instruction, updates values or control flow, and proceeds to the next instruction. A call opcode can transfer control to another op array or an internal function; a jump changes the next instruction; a return ends the current frame.

This is a conceptual model. The concrete `zend_op` layout and operand encoding are defined by the PHP build and can change.

## Core Concept

For this source:

```php
<?php

$total = $left + $right;
```

the compiler needs to obtain two values, perform addition, and assign the result. An illustrative sequence might be:

```text
FETCH_R       $left       → temporary-1
FETCH_R       $right      → temporary-2
ADD           temporary-1, temporary-2 → temporary-3
ASSIGN        temporary-3 → $total
```

The names and exact sequence are illustrative. Modern PHP can use specialized instructions, different operand modes, optimizer transformations, or fused call paths. The stable insight is that source expressions become explicit reads, operations, writes, and control-flow operations.

## How It Works

The opcode catalogue is generated and maintained in php-src. [Zend/zend_vm_opcodes.h](https://github.com/php/php-src/blob/master/Zend/zend_vm_opcodes.h) defines opcode names and IDs for a particular source tree. The [php-src contributor guide](https://github.com/php/php-src/blob/master/CONTRIBUTING.md) identifies generated opcode files and the generator involved. Do not copy numeric opcode IDs into a long-lived tool without pinning it to a PHP version and build.

An opcode has operands with modes. Current compile headers describe modes such as:

- `IS_CONST`: a literal or constant operand;
- `IS_CV`: a compiled variable slot;
- `IS_TMP_VAR`: a temporary value;
- `IS_VAR`: a variable-like storage operand.

These names are internal vocabulary. They help explain why assigning a value, reading an array dimension, and passing by reference require different work. Later chapters cover zvals, references, and copy-on-write in more detail.

Control flow is explicit. A source `if` can compile into a conditional jump over one branch and an unconditional jump over the other. A `foreach` needs initialization, fetch, body, and cleanup instructions. A `try/catch` needs exception metadata and handler instructions in addition to the ordinary path. The VM must preserve language-defined evaluation order and cleanup behavior.

## What PHP Does

The language defines observable results, evaluation order where specified, error behavior, and side effects. It does not promise a particular number or order of internal opcodes. Two PHP versions can produce different opcode arrays while producing the same valid program result. Code that relies on an opcode dump is therefore diagnostic or version-pinned tooling, not ordinary application code.

## What Zend Does

The current [Zend compile header](https://github.com/php/php-src/blob/master/Zend/zend_compile.h) declares `zend_op` fields including opcode, operands, result, line number, and types, along with `zend_op_array` storage for the instruction buffer, literals, compiled variables, exception regions, and runtime cache. VM execution is generated from templates and opcode definitions; the generated executor is not the same thing as the opcode list.

The handler for an opcode can be selected or specialized by the VM build. The handler may invoke core routines, extension functions, memory management, object handlers, or userland calls. Therefore an opcode dump alone does not tell you the total cost of an operation.

## Minimal Example

```php
<?php

declare(strict_types=1);

function label(string $name): string
{
    return 'User: ' . $name;
}

echo label('Ada');
```

An illustrative opcode-shaped outline is:

```text
function label:
    RECV        parameter $name
    CONCAT      'User: ', $name → temporary
    RETURN      temporary

top-level file:
    INIT_FCALL  label
    SEND_VAL    'Ada'
    DO_FCALL    → temporary
    ECHO        temporary
    RETURN      1
```

This outline is deliberately not a claimed dump from a particular PHP binary. It shows the categories of work: receive arguments, construct a value, return it, initialize a call, send an argument, perform the call, and emit output.

## Practical Example: Inspecting Opcode Output

With a development build of OPcache, the documented `opcache.opt_debug_level` setting can emit opcode dumps for debugging. The PHP manual documents `0x10000` for compiler-produced opcodes before optimization and `0x20000` for optimized code ([OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)). Treat this as a diagnostic facility:

```bash
php -d opcache.enable_cli=1 \
    -d opcache.opt_debug_level=0x10000 \
    -r 'echo 1 + 2, PHP_EOL;'
```

Exact output depends on PHP version, build, enabled extensions, and OPcache configuration. Do not enable verbose opcode dumps in a production environment casually; they can disclose source details and create substantial diagnostic output.

A third-party tool such as VLD may offer a different disassembly view, but it is not part of the PHP core contract. Pin it with the PHP version and verify that it supports the exact binary under test.

## Bad Example

```php
// “This is fast because it emits exactly three opcodes on my laptop.”
$result = $a + $b;
```

Even if a particular build shows three instructions, operand values, zval types, overloaded objects, references, error paths, cache state, CPU behavior, and surrounding calls can dominate runtime. A database or network call can make the instruction count irrelevant.

## Better Example

Form a measurable hypothesis:

```text
symptom: a hot loop consumes CPU
    → profile the loop under production-like data
    → check whether time is in VM dispatch, array/hash operations, calls, or I/O
    → compare an algorithm/data-structure change
    → benchmark cold and warm runs with memory measurements
```

Opcode output can explain a surprising result, such as repeated function calls or unnecessary conversions. It should support a profile, not replace one.

## Edge Cases

- An optimizer can remove, combine, reorder where semantics permit, or specialize work; “before optimizer” and “after optimizer” can differ.
- A function call may use different call opcodes depending on whether the target is internal, userland, known by name, dynamic, or a method.
- A jump target is an instruction location in the compiled unit, not a source line.
- One source line can generate many opcodes, and one opcode can correspond to a larger operation.
- Exceptions transfer control through handler metadata and cleanup paths rather than simply returning an error value.
- Generators and fibers suspend and resume execution with state that is not captured by a single linear trace.
- Extension functions execute through internal-function boundaries that may not resemble userland opcode sequences.

## Performance

Opcode dispatch has overhead, but reducing opcode count is not a universal optimization. Cost can come from operand fetches, temporary allocation, reference separation, hash lookups, object handlers, internal calls, memory allocation, and cache misses. The VM may have specialized handlers, and OPcache may make a warm request materially different from a cold one.

When comparing alternatives, record:

```text
wall time, CPU time, allocations/peak memory, request SAPI,
PHP version/build, OPcache state, input size, and external I/O
```

An O(n) algorithm with fewer opcodes can still lose to an O(log n) design at scale, while a small in-process loop can beat a network round trip for tiny inputs. Complexity and measurement belong together.

## Security

Opcode dumps can expose source-derived names, literals, file paths, and control-flow details. Restrict them to development or controlled diagnostics. Do not use opcode inspection as an authorization mechanism or as proof that untrusted code is safe. A dangerous operation can be hidden behind a call, an extension, dynamic dispatch, or an included file.

## Testing

Application tests should assert behavior, not private opcode sequences. Opcode-level tests are appropriate for:

1. Zend Engine changes.
2. A profiler or tracing extension pinned to a PHP build.
3. A performance regression experiment with a documented environment.
4. A debugging tutorial that clearly labels output as version-specific.

If you snapshot opcode dumps, normalize file paths and decide whether the snapshot represents pre-optimization or post-optimization output. Update snapshots deliberately when moving PHP versions; a changed dump is not automatically a behavior regression.

## Common Mistakes

- Treating opcode numbers as stable across PHP versions.
- Assuming opcode count equals runtime cost.
- Confusing an opcode with a CPU instruction.
- Ignoring optimizer and OPcache state.
- Publishing production opcode dumps that reveal source or configuration details.
- Using an unofficial disassembler as if it defined Zend’s public API.

## Senior Engineer Thinking

Opcodes are a microscope, not a map of the whole system. Use them after locating the hot path and identifying the boundary involved. If the profile says the process waits on PostgreSQL, opcode trivia is not the bottleneck. If a tight loop spends time in repeated hash lookups and conversions, opcode and VM knowledge can help explain why an algorithm or data-structure change matters.

## Exercises

1. Draw the conceptual opcodes for a short-circuit `&&` expression and identify its jump target.
2. Compare the source-level work and likely VM categories for a direct function call and a dynamic callable.
3. Produce a pre-optimization opcode dump in a development environment, then repeat with optimization enabled. Document the PHP version and configuration.
4. Choose a real slow function from a benchmark and determine whether its dominant cost is VM work, allocation, extension code, or I/O.

## Review Questions

1. What information does an opcode carry beyond its operation name?
2. Why can two PHP versions emit different opcodes for equivalent behavior?
3. Why is opcode count an incomplete performance metric?
4. What is the difference between an opcode and a native CPU instruction?
5. When is an opcode snapshot a legitimate test, and when is it brittle?

## Summary

Opcodes are Zend VM instructions stored in version-specific op arrays. They make reads, writes, calls, jumps, cleanup, and results explicit, but they are not a stable application API or native CPU instructions. Use documented opcode diagnostics and php-src headers to investigate hypotheses, then validate conclusions with production-like profiling and behavior tests.

## References

- [php-src: `Zend/zend_vm_opcodes.h`](https://github.com/php/php-src/blob/master/Zend/zend_vm_opcodes.h)
- [php-src: `Zend/zend_compile.h`](https://github.com/php/php-src/blob/master/Zend/zend_compile.h)
- [php-src: `Zend/zend_opcode.c`](https://github.com/php/php-src/blob/master/Zend/zend_opcode.c)
- [php-src: `Zend/zend_vm_execute.skl`](https://github.com/php/php-src/blob/master/Zend/zend_vm_execute.skl)
- [php-src: contributor guide](https://github.com/php/php-src/blob/master/CONTRIBUTING.md)
- [PHP Manual: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
