---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 40
title: Source Code to Execution
slug: source-code-to-execution
status: complete
summary: ../../_ai/chapter-summaries/040-source-code-to-execution-summary.md
---

# Chapter 40 — Source Code to Execution

## Why This Matters

When a PHP program is slow, fails before its first log line, or behaves differently after enabling OPcache, “PHP runs the file” is not a useful enough explanation. A source file passes through several representations before user code produces a result:

```text
bytes on disk
    ↓
characters and lexical tokens
    ↓
syntax tree (AST)
    ↓
compiled function/file structures and opcodes
    ↓
Zend VM execution
    ↓
side effects and a return value
```

This is a reasoning model, not a promise that every build exposes each phase as a separate durable object. The model helps us put an observation in the right layer. A missing semicolon is a parsing failure; an undefined array key is generally an execution-time behavior; a slow database call is not fixed by changing an opcode.

## Mental Model

For a normal script, the engine opens a file, scans PHP and non-PHP regions, parses PHP syntax, compiles the resulting structure, and executes the resulting op array. “Compile” here usually means compiling to Zend VM instructions, not compiling PHP directly to native machine code. Later, the VM dispatches those instructions and calls extension or userland implementations as needed.

The complete application has more boundaries than this language pipeline:

```text
web server / CLI
    → SAPI and configuration
    → Zend Engine startup
    → compile one file (or obtain a cached result)
    → execute top-level code
    → include/autoload more files as requested
    → return, emit output, and clean up request state
```

The first file is not necessarily the only file compiled. An `include`, `require`, autoloader, or `eval()` can introduce more source later. A file included conditionally may never be compiled during a particular request.

## Core Concept

Keep four distinctions separate:

1. **Source code** is text with delimiters, comments, whitespace, and spelling.
2. **Tokens** are scanner output such as `T_VARIABLE`, `T_STRING`, or a literal `;`.
3. **An AST** records grammatical structure without preserving every spelling detail.
4. **Opcodes** are executable VM instructions with operands, results, metadata, and control-flow targets.

Each representation answers a different question. A formatter needs source locations and tokens. A static analyzer needs an AST plus name and type analysis. A profiler often cares about executed functions and opcodes. The engine does not need to preserve comments in the same way an editor does.

The [PHP 7 AST RFC](https://wiki.php.net/rfc/abstract_syntax_tree) is an important historical boundary: PHP 7 replaced the older parser-to-opcode path with an intermediate AST. This decoupling made the parser and compiler easier to evolve, but the AST remains an engine implementation detail rather than a stable application API.

## How It Works

Consider this small file:

```php
<?php

declare(strict_types=1);

function subtotal(int $quantity, int $unitPrice): int
{
    return $quantity * $unitPrice;
}

echo subtotal(3, 120);
```

Conceptually, the stages do the following:

- The scanner recognizes the open tag, declaration, keywords, identifiers, literals, operators, and punctuation.
- The parser recognizes a function declaration, parameter declarations, a return statement, a multiplication expression, and a call expression.
- The compiler creates a function representation for `subtotal` and instructions for multiplication, the call, and output.
- The executor obtains values for `3` and `120`, invokes the function, computes `360`, and sends that value to the output mechanism.

The engine may resolve some names or evaluate some constant expressions during compilation. It does not execute the multiplication while parsing. Conversely, compiling a function body does not call the function. A declaration can be compiled and registered before a later top-level statement invokes it.

## What PHP Does

PHP defines the observable language rules: what syntax is valid, how operators associate, what a type declaration means, which errors are reported, and what a program can observe. The command-line `-l` option asks PHP to perform a syntax check without executing the supplied file; it is therefore useful for testing the source-to-parse part of the pipeline, not for proving that a program is safe or correct ([command-line options](https://www.php.net/manual/en/features.commandline.options.php)).

PHP source can also contain literal text outside PHP tags. That text is part of the source file, but its output is an execution effect. It is not printed while the lexer is merely reading the file.

## What Zend Does

In current php-src, the relevant implementation pieces include the scanner and parser, `zend_ast` structures, compiler entry points, `zend_op_array`, and the VM. The public-facing names are useful orientation, but their layouts and generated files can change between PHP releases. The [Zend compile declarations](https://github.com/php/php-src/blob/master/Zend/zend_compile.h) include entry points such as `compile_file()`, `compile_string()`, and `zend_compile_ast()`, as well as the `zend_op_array` representation.

The normal path is not “the PHP interpreter repeatedly reparses every expression.” It is closer to “produce an executable representation, then dispatch it.” OPcache can retain and optimize compiled results across requests when enabled and correctly configured. That changes how often work is done, not the language meaning of a valid program. See the [OPcache configuration documentation](https://www.php.net/manual/en/opcache.configuration.php) and Chapter 59 for operational details.

## Minimal Example

Compare a syntax failure with a runtime failure:

```php
<?php

echo "before\n";

// Parse error: the statement is incomplete.
if (true {
    echo "never reached\n";
}
```

```php
<?php

echo "before\n";
throw new RuntimeException('failure during execution');
echo "after\n";
```

The first file cannot be compiled as a valid unit, so its body does not execute. The second file can be parsed and compiled; execution prints `before` and then throws. In a web request, framework bootstrap or an earlier included file may already have run, so “the file did not execute” must not be generalized to “nothing in the request executed.”

## Practical Example: Observing the Boundaries

Use three separate checks during diagnosis:

```bash
php -l src/Price.php
php -d display_errors=1 src/Price.php
php -i | grep -E 'opcache.enable|opcache.validate_timestamps'
```

The first asks about syntax. The second exercises runtime behavior in a CLI environment. The third inspects configuration, not application correctness. In CI, syntax checks complement unit and integration tests; they do not replace them.

## Production Example

Suppose a deployment writes new PHP files in place while PHP-FPM workers are serving traffic. A worker may read an incomplete file, fail during scanning or parsing, or continue using a cached version depending on deployment and OPcache settings. A safer deployment writes a complete release into a new directory and atomically changes the application path or symlink, then handles cache invalidation according to the runtime configuration. The important lesson is that source visibility, compilation, and cache freshness are operational boundaries.

## Bad Example

```php
// “It passed php -l, so it is ready.”
php_lint_passed();
```

Syntax checking cannot detect a wrong SQL query, a missing environment variable, an authorization bug, a timeout, a race, or a domain invariant violated at runtime. It also cannot prove that every dynamically loaded file is available in production.

## Better Example

Treat each boundary as a testable claim:

```text
source is syntactically valid
    → dependencies can be loaded
    → configuration is valid
    → behavior is correct for known inputs
    → side effects are authorized and durable
    → latency and memory fit the deployment budget
```

This sequence also improves incident reports. “The request failed” becomes “the main file parsed, the autoloader loaded the class, the database call timed out, and the retry returned a duplicate response,” which points to a different remedy.

## Edge Cases

- `include` and `require` compile another file when execution reaches the statement; a syntax error in a conditional include can therefore appear only when that branch is taken.
- `eval()` compiles a string in the current execution context. It adds complexity to observability, caching, and security, and should rarely be necessary.
- A syntax error in one file does not imply that every file in an application is invalid. Compilation is generally per file, although startup configuration and included dependencies create additional failure paths.
- `php -l` checks syntax but does not run autoloaders or execute expressions.
- An extension can participate in compilation and execution through Zend extension hooks; behavior outside core should be attributed to the extension, not vaguely to “PHP.”

## Performance

For a short-lived request, source scanning and compilation are often small compared with database and network latency, but they are not free. Large dependency graphs increase file I/O, scanning, allocations, and compilation work. OPcache can reduce repeated compilation and may optimize opcode arrays. Measure with the same SAPI and configuration as production; CLI timings do not automatically predict PHP-FPM behavior.

## Security

Source code is executable input to the deployment system. Prevent untrusted data from reaching `eval()`, dynamic includes, or code-generation paths. Keep production errors from exposing source paths and stack traces to clients, while retaining structured logs for operators. A valid parse is not a security decision: validation, authorization, safe query construction, and output encoding happen at later application boundaries.

## Testing

A useful small test matrix includes:

1. Run `php -l` over every tracked PHP file.
2. Execute unit tests with production-like PHP version and extensions.
3. Test the autoloader and configuration in a clean process.
4. Exercise a representative request through the real SAPI or container image.
5. If OPcache settings matter, test a deployment and restart/invalidation procedure rather than only a warm CLI run.

Keep a failing fixture for a known syntax error and a separate fixture for a runtime exception. The distinction is valuable when teaching tooling and diagnosing CI failures.

## Common Mistakes

- Calling Zend VM “a compiler to machine code” in a normal PHP build.
- Treating the token stream, AST, and opcode array as interchangeable.
- Assuming `php -l` catches runtime failures.
- Assuming every included file is compiled at application startup.
- Debugging OPcache by changing source code randomly instead of inspecting timestamps, resets, and deployment atomicity.
- Describing implementation details as language guarantees.

## Senior Engineer Thinking

When a symptom appears, ask which representation must already exist for the symptom to occur. A parse error means execution of that file has not begun. A VM warning means compilation succeeded far enough to produce executable structures. A database timeout means the program crossed the language/runtime boundary and performed I/O. This vocabulary prevents expensive debugging in the wrong layer.

## Exercises

1. Create one file with a parse error and one with a thrown exception. Compare `php -l` and normal execution.
2. Add an `include` inside an `if` branch and put a syntax error in the included file. Verify when the error appears.
3. Run the same script with OPcache disabled and enabled in a controlled environment. Record what changes and what does not.
4. Draw the source-to-execution pipeline for a request that autoloads one class and performs one database query. Mark each process and network boundary.

## Review Questions

1. Why is an opcode array not the same thing as native machine code?
2. What can `php -l` establish, and what can it not establish?
3. Why can an included file fail to compile only on a particular request?
4. Which parts of the pipeline are language behavior and which are Zend implementation details?
5. How would an atomic deployment reduce source/cache inconsistency?

## Summary

PHP source moves through scanning, parsing, AST construction, compilation, and VM execution. The stages are conceptually distinct even when implementation optimizations combine or cache them. Use the distinction to classify failures, performance costs, tests, and security controls. The next chapters examine the first stages in detail: lexical scanning, parsing, and the AST.

## References

- [PHP Manual: `token_get_all()`](https://www.php.net/manual/en/function.token-get-all.php)
- [PHP Manual: command-line options](https://www.php.net/manual/en/features.commandline.options.php)
- [PHP RFC: Abstract Syntax Tree](https://wiki.php.net/rfc/abstract_syntax_tree)
- [php-src: `Zend/zend_compile.h`](https://github.com/php/php-src/blob/master/Zend/zend_compile.h)
- [PHP Manual: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
