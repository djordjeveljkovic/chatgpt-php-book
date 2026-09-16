---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 44
title: Compilation
slug: compilation
status: complete
summary: ../../_ai/chapter-summaries/044-compilation-summary.md
---

# Chapter 44 — Compilation

## Why This Matters

Compilation is where parsed meaning becomes an executable plan for the Zend VM. It is not merely a syntax check and it is not normally native-code generation. The compiler decides how expressions, variables, declarations, branches, calls, exceptions, and source locations are represented in an op array.

This matters when a language feature has a compile-time restriction, when a debugger shows an unexpected line, when a supposedly simple expression allocates temporaries, or when OPcache reports a different opcode sequence from a cold compilation. Understanding compilation gives you a precise place to ask “what was decided before this value existed?”

## Mental Model

```text
AST
  ↓ compiler traversal and context
zend_op_array
  ├── zend_op instructions
  ├── literals and compiled variables
  ├── function/class metadata
  ├── jump and exception information
  └── source and runtime-cache metadata
  ↓
Zend VM execution
```

An op array is a compiled function-like unit. A file has a top-level op array; user functions and methods have their own op arrays; closures have executable function data plus captured context. The exact ownership and fields are internal and change over time, but this decomposition is a useful current model.

## Core Concept

Compilation chooses instructions without knowing every runtime value. For:

```php
<?php

$taxed = $subtotal * 1.2;
```

the compiler can represent a multiplication and assignment. It cannot generally replace `$subtotal` with a number because that value is supplied during execution. For a constant expression, the compiler or a later optimizer may precompute more, subject to language rules and side-effect constraints.

Compilation also creates metadata needed later. A function’s parameters, return type, flags, line information, exception regions, literal values, and local-variable slots are not all “instructions,” but execution and reflection may need them.

## How It Works

The compiler walks AST nodes in context. Context includes whether an expression is read, written, passed by reference, used as a condition, inside a loop, inside a function, or inside a class. The same syntax can therefore emit different instruction patterns depending on where it appears.

For a call:

```php
<?php

$result = calculate($input, 10);
```

an illustrative sequence is:

```text
FETCH $input
prepare call to calculate
send value $input
send constant 10
perform call
assign returned value to $result
```

The actual opcode names, operand kinds, and call fast paths are version-specific. The important point is that the compiler turns nested syntax into an ordered instruction sequence with explicit data movement and control-flow edges.

Branches become jumps:

```php
<?php

if ($enabled) {
    echo 'on';
} else {
    echo 'off';
}
```

Conceptually:

```text
evaluate $enabled
jump if false → else
echo 'on'
jump → end
else:
echo 'off'
end:
```

The compiler must patch jump destinations after it knows where blocks end. Loops, `break`, `continue`, `match`, `try/catch`, and `finally` add more control-flow bookkeeping.

## What PHP Does

The language exposes some compile-time effects and failures. A malformed declaration, invalid syntax, or forbidden construct can stop compilation. Other failures are deliberately deferred until execution because they depend on runtime values, loaded symbols, permissions, or external state.

PHP’s `php -l` command exercises syntax checking without executing the file. It is a useful compile-front-end smoke test, but it does not demonstrate that the compiler can resolve every runtime dependency or that the VM will successfully execute every path.

## What Zend Does

Current php-src declares compiler entry points such as `compile_file()`, `compile_string()`, and `zend_compile_ast()` in [Zend/zend_compile.h](https://github.com/php/php-src/blob/master/Zend/zend_compile.h). The compiler implementation is primarily [Zend/zend_compile.c](https://github.com/php/php-src/blob/master/Zend/zend_compile.c). The `zend_op_array` structure includes an opcode buffer, literals, compiled-variable names, line bounds, exception metadata, static variables, and runtime-cache information. These are implementation details of the selected PHP build, not a stable extension ABI for userland code.

Compilation can invoke extension hooks and can create structures for classes and functions. OPcache may then optimize or cache the compiled representation. “The compiler did it” should therefore be narrowed to a specific phase: core compilation, extension hook, optimizer pass, or runtime lookup.

## Minimal Example

```php
<?php

declare(strict_types=1);

function add(int $left, int $right): int
{
    return $left + $right;
}

echo add(2, 3);
```

The function declaration produces a function representation; it does not execute `left + right` at file compilation time. The top-level call compiles to call-related instructions and executes only when the VM reaches it. Parameter and return type enforcement are part of the language’s execution semantics, even though the compiler records the declarations.

## Practical Example: Compile-Time and Runtime Claims

```php
<?php

const DEFAULT_LIMIT = 25;

function firstPage(array $rows, int $limit = DEFAULT_LIMIT): array
{
    return array_slice($rows, 0, $limit);
}
```

The compiler can record the declaration and constant expression information. It cannot know the contents of `$rows` or whether the caller will provide a valid value for `$limit`. It also cannot prove that `array_slice()` will be available if the runtime has been built without the required extension or that the call will not be interrupted by a resource failure.

## Bad Example

```php
// “The compiler will optimize this database query.”
$rows = $pdo->query($sql)->fetchAll();
```

PHP compilation can optimize or specialize internal instruction handling, but it does not inspect the database’s schema and produce an SQL query plan. SQL parsing, indexing, locking, and execution belong to the database. Keep the PHP compiler boundary separate from the database optimizer boundary.

## Better Example

```text
PHP compiler: source structure → VM instructions
PHP optimizer/OPcache: safe transformations and caching of compiled code
database: SQL text/plan → rows and locks
application: map rows → domain behavior and response
```

Measure each boundary separately. A function that spends 2 ms compiling but 400 ms waiting on SQL needs a database investigation, not an opcode micro-optimization.

## Edge Cases

- A function body can be compiled without being called.
- A dynamically included file is compiled when the include operation executes.
- `eval()` compiles a string at runtime and can generate an op array with a different lifetime and context from a file.
- Constant folding is constrained by semantics; do not assume every expression is folded.
- Name resolution can use namespace and import context known while compiling, while autoloading and dynamic names can defer work.
- `try`, `catch`, and `finally` require exception-region metadata as well as ordinary jumps.
- Closures need capture information, and generators/fibers require resumable execution state beyond a simple linear function call.

## Performance

Compilation cost includes file I/O, scanning, AST allocation, opcode allocation, symbol/declaration work, and possibly optimization. It is paid repeatedly in a cold short-lived process unless a cache or persistent worker reuses results. Opcode count is only a proxy: a single call can perform expensive I/O, while many simple instructions may be cheap. Benchmark warm and cold paths, and include memory and cache invalidation behavior.

The compiler also creates temporary values and metadata that affect peak memory. Rewriting code to reduce “lines” does not necessarily reduce instructions or allocations; inspect behavior rather than trusting source brevity.

## Security

Compilation is not a sandbox. If an attacker controls PHP source, a successful compile only means the source has legal structure; execution can still perform dangerous effects. Restrict code loading, file permissions, dynamic includes, and deployment inputs. Be cautious with tools that compile or inspect source supplied by users, because parser and compiler resource consumption can itself be a denial-of-service vector.

## Testing

Test compilation as one layer and execution as another:

1. Lint all production source in CI.
2. Run tests on the minimum and current supported PHP versions.
3. Include fixtures for syntax and declaration failures.
4. Execute branches that exercise dynamic includes, closures, exceptions, generators, and type checks when your application uses them.
5. If opcode output is part of a profiler or extension test, pin the PHP build and treat instruction details as version-specific.

For a compiler-facing tool, retain a source fixture, normalized AST expectation, and behavioral expectation. This helps distinguish a changed tree from a changed optimization.

## Common Mistakes

- Equating compilation with native machine-code generation.
- Assuming compilation evaluates all expressions.
- Assuming a successful compile proves dependencies, configuration, or permissions.
- Treating op-array fields as a stable public API.
- Optimizing instruction count before measuring I/O and database latency.
- Ignoring the difference between cold compilation and cached execution.

## Senior Engineer Thinking

Compilation is a contract between language meaning and runtime execution. When a feature behaves unexpectedly, ask what information was available at compile time and what was necessarily deferred. When performance changes, ask whether the change affected source parsing, compilation, optimizer output, VM dispatch, a function call, or an external boundary.

## Exercises

1. Draw the conceptual instruction sequence for an `if/else` and a `while` loop.
2. Mark which parts of `array_slice($rows, 0, $limit)` are known during compilation and which require runtime values.
3. Compare a cold PHP process with a warm OPcache process and record startup time without claiming that the result generalizes to all servers.
4. Find a source construct in your application whose compilation creates nested function or closure metadata. Explain its lifetime.

## Review Questions

1. What is an op array, conceptually?
2. Why does a compiler emit jumps for branches?
3. Which failures can syntax checking catch, and which require execution?
4. Why is PHP compilation not the same as SQL planning?
5. What makes opcode count an incomplete performance metric?

## Summary

Compilation traverses AST structure and produces executable Zend structures, especially op arrays containing instructions and metadata. It records what can be known from source and defers value-dependent behavior to execution. The compiler, OPcache optimizer, VM, and external systems are distinct layers; accurate diagnosis depends on keeping them distinct.

## References

- [php-src: `Zend/zend_compile.c`](https://github.com/php/php-src/blob/master/Zend/zend_compile.c)
- [php-src: `Zend/zend_compile.h`](https://github.com/php/php-src/blob/master/Zend/zend_compile.h)
- [PHP Manual: command-line options](https://www.php.net/manual/en/features.commandline.options.php)
- [PHP RFC: Abstract Syntax Tree](https://wiki.php.net/rfc/abstract_syntax_tree)
- [PHP Manual: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
