---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 8
title: Syntax
slug: syntax
status: complete
summary: ../../_ai/chapter-summaries/008-syntax-summary.md
---

# Chapter 8 — Syntax

## Why This Matters

Syntax is the part of PHP that the parser must understand before the program can run. That sounds elementary, but syntax is more than punctuation. It is the structure that tells PHP which text is code, which text is output, which values belong to an expression, and where a block begins and ends.

A syntax error happens before the application gets to make a useful decision. No validation branch, exception handler, log statement, or database transaction inside the malformed file can repair it. A missing brace can therefore take down an entire entry point, while a misplaced semicolon can change a condition into a statement that always runs.

The practical goal is not to memorize every grammar rule. It is to learn to see PHP source as a small language with boundaries:

```text
source file
  → PHP and non-PHP regions
  → tokens and literals
  → expressions
  → statements and blocks
  → parsed program
  → execution
```

That model makes syntax errors easier to locate and helps us choose source structures that remain readable under change.

## Mental Model

PHP source contains several different things:

| Element | Meaning | Example |
| --- | --- | --- |
| Token | A grammar unit recognized by the lexer | `function`, `(`, `+`, `TennisCourt` |
| Literal | A value written directly in source | `42`, `true`, `'clay'` |
| Expression | Code that produces a value | `$price * $quantity` |
| Statement | An instruction or declaration | `$total = 42;`, `return $total;` |
| Block | Statements grouped under a construct | `{ ... }` |
| File boundary | A transition between PHP code and literal output | `<?php ... ?>` |

An expression can appear inside a statement. A statement can appear inside a block. A token is not itself a runtime value: the token `42` becomes an integer value when the parsed program executes it.

This distinction matters when diagnosing failures. A malformed expression is a parse-time problem. A valid expression that divides by zero, calls an unavailable dependency, or violates a type declaration is an execution-time problem. Chapter 4 introduced this source-to-execution boundary; this chapter focuses on the source side of it.

## Core Concept

### PHP code has explicit entry points

A pure PHP file normally starts with the `<?php` opening tag. The short echo tag, `<?=`, is available for output and does not require a separate `<?php` before it:

```php
<?php

declare(strict_types=1);

echo 'Rendered by PHP';
```

For application source files, omit the closing `?>` tag. This prevents accidental whitespace or a byte-order mark from becoming output before headers are sent. A template can close a PHP block because it intentionally alternates between code and markup, but a library file should normally remain PHP-only.

Everything outside PHP tags is literal output when a file is used as a template. It is not a comment and it is not automatically escaped:

```php
<h1><?= htmlspecialchars($title, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></h1>
```

The output boundary is a security boundary. `<?= $title ?>` is only safe when the value is already trusted for that exact HTML context. Syntax makes output possible; it does not make output safe.

### Statements end and blocks nest

Most PHP instructions end with a semicolon. Braces group statements for functions, conditionals, loops, classes, and namespaces. Indentation helps humans, but braces and tokens determine the program:

```php
<?php

if ($requestedSlots > 0) {
    $message = 'Slots requested';
} else {
    $message = 'No slots requested';
}

echo $message;
```

The closing PHP tag at the end of a block implies the final semicolon, but relying on that exception makes code harder to move and lint. Use semicolons consistently in PHP-only source.

### Expressions produce values

Assignments, function calls, comparisons, array literals, object creation, and arithmetic are expressions or contain expressions. A statement usually uses their result:

```php
$subtotal = $unitPrice * $quantity;
$label = trim($courtName);
$reservation = new Reservation($courtId, $startsAt, $endsAt);
```

Do not read precedence from visual intuition. Parenthesize a business decision when the grouping matters, even when PHP’s precedence rules would produce the same result today. Chapter 13 covers operators in detail; here the important rule is that punctuation is not decoration. `.` concatenation, `+` addition, `=` assignment, `===` strict comparison, and `=>` an array entry all have different grammar and meaning.

### Names are part of the contract

Variables begin with `$`; class, interface, trait, enum, function, and constant names follow their own grammar rules. Namespaces qualify symbols so two parts of an application can use the same short class name without being the same class:

```php
<?php

declare(strict_types=1);

namespace App\Reservations;

final class CourtId
{
    public function __construct(public readonly int $value)
    {
    }
}
```

The namespace declaration must occur at the top level of the file, before ordinary executable code. Imports with `use` are aliases resolved by the parser/compiler; they do not instantiate anything and do not load a class by themselves.

Names should tell the reader what a value means. `$x` may be acceptable as a short loop coordinate. `$reservationStart` is better when the value crosses a boundary or participates in a rule. Naming is a low-cost form of documentation, not a replacement for types or validation.

## How It Works

The Zend Engine first needs a valid representation of the source. Conceptually, the process is:

```text
PHP source
  → lexical tokens
  → grammar/parser
  → compiled representation
  → execution
```

The lexer recognizes pieces such as identifiers, keywords, numbers, strings, and punctuation. The parser checks that those pieces form valid PHP. Only after that can the runtime execute the resulting program. The exact internal representation is developed in Volume IV; the useful boundary here is that parse errors happen before normal execution.

For example, this file cannot run far enough to print the message:

```php
<?php

if ($isOpen {
    echo 'Court available';
}
```

The missing `)` is a structural error. By contrast, this file is syntactically valid but can fail while running:

```php
<?php

echo $reservations[0]['court_id'];
```

If the array is empty or has a different shape, execution may emit a warning or throw depending on the exact operation and PHP version. The parser cannot know the data shape. Syntax validates arrangement; it does not validate runtime facts.

## What PHP Does

PHP provides several source forms that are useful in different environments:

- PHP-only files for application code and libraries;
- mixed PHP/HTML templates for presentation;
- CLI scripts with the same language syntax but a different entry point;
- attributes such as `#[Route('/courts')]`, which attach structured metadata to declarations;
- anonymous functions and arrow functions for values that represent behavior;
- alternative constructs such as `match`, named arguments, and the null-coalescing operator in modern PHP.

These forms are language features, not framework magic. A framework may inspect attributes, invoke a callable, or choose a controller based on a route. PHP itself only parses the declaration and makes the resulting metadata or callable available.

Keep syntax appropriate to the supported PHP version. A project targeting PHP 8.0 cannot use syntax added later merely because the developer’s local PHP is newer. The `composer.json` platform setting, CI runtime, deployment runtime, and static-analysis configuration should agree on the version contract.

## What Zend Does

Zend turns valid source into executable structures and then evaluates expressions, calls functions, manages values, and propagates failures. The source punctuation is not interpreted one character at a time on every operation in the simplistic sense; compilation and optional OPcache change the path. The important engineering consequence is that a parse error prevents the entry point from reaching application code, while an execution error can occur after side effects have started.

The runtime also distinguishes code from literal template output. When a template leaves PHP mode, the bytes outside the tags become output unless the template logic skips them. That output may be buffered, sent through a response object, or written directly depending on the environment. Treating “a PHP file” as only executable code is therefore an incomplete model.

## Minimal Example

Here is a complete PHP-only program with a namespace, a typed function, an expression, and an explicit output boundary:

```php
<?php

declare(strict_types=1);

namespace App\Demo;

function formatCourtLabel(string $surface, int $courtNumber): string
{
    return "Court {$courtNumber} ({$surface})";
}

$label = formatCourtLabel('clay', 3);

echo $label, PHP_EOL;
```

The string uses interpolation for simple scalar expressions. Curly braces make the variable boundary obvious inside a larger string. If the expression becomes complex, calculate it separately instead of turning the string into a miniature program.

## Practical Example

Suppose a CLI command accepts a court number and a surface and needs to render a stable line. Keep parsing, validation, and formatting visibly separate:

```php
<?php

declare(strict_types=1);

function parseCourtNumber(string $raw): int
{
    $number = filter_var($raw, FILTER_VALIDATE_INT);

    if ($number === false || $number < 1) {
        throw new InvalidArgumentException('Court number must be a positive integer.');
    }

    return $number;
}

function courtLabel(int $number, string $surface): string
{
    $normalizedSurface = strtolower(trim($surface));

    if (!in_array($normalizedSurface, ['clay', 'hard', 'grass'], true)) {
        throw new InvalidArgumentException('Unknown court surface.');
    }

    return sprintf('Court %d (%s)', $number, $normalizedSurface);
}

$number = parseCourtNumber($argv[1] ?? '');
$surface = $argv[2] ?? '';

echo courtLabel($number, $surface), PHP_EOL;
```

The syntax is ordinary, but the structure communicates the control flow. The input boundary is visible, the validator returns a value with a known type, and the formatter does not reach into `$argv` itself. That separation lets a web adapter reuse the formatting rule without pretending that HTTP and CLI inputs are identical.

## Production Example

A production application usually has PHP-only source files and a deliberate rendering boundary. A minimal template might look like this:

```php
<?php
/** @var list<array{number: int, surface: string}> $courts */
?>
<ul>
<?php foreach ($courts as $court): ?>
    <li>
        Court <?= htmlspecialchars((string) $court['number'], ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        (<?= htmlspecialchars($court['surface'], ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>)
    </li>
<?php endforeach; ?>
</ul>
```

The short echo tag makes the output location clear. `htmlspecialchars()` is used for an HTML text context; another context, such as an HTML attribute, JavaScript string, SQL statement, or URL, needs its own correct encoding or parameterization strategy. A template’s syntax does not select the correct defense automatically.

## Bad Example

This file mixes syntax tricks with hidden behavior:

```php
<?php

if ($court && $court['open'] = true)
    echo 'Available';
else
    echo 'Closed';
```

The assignment inside the condition changes state and then evaluates as a value. It may make the branch appear to work while silently overwriting `open`. The missing braces make a later edit dangerous: adding an extra statement with the indentation of the branch does not put it in the branch.

Another common failure is a closing tag followed by invisible whitespace in a library file:

```php
<?php

return require __DIR__ . '/config.php';
?>
```

If the file contains trailing output, an HTTP response may already have begun before code tries to send headers. The syntax is legal; the file boundary is operationally wrong for a library.

## Better Example

Make the condition a comparison, use braces, and isolate the decision:

```php
<?php

declare(strict_types=1);

$isOpen = $court['open'] ?? false;

if ($isOpen === true) {
    echo 'Available';
} else {
    echo 'Closed';
}
```

In application code, prefer a typed value or a validated DTO before this point. The explicit comparison is useful when the domain specifically requires the boolean `true`; if any truthy value is valid, the condition can be simpler. The choice should express the contract rather than follow a universal style rule.

## Edge Cases

- A UTF-8 byte-order mark or whitespace before `<?php` is output, not ignored source. It can cause “headers already sent” failures.
- A closing PHP tag is optional at the end of a PHP-only file. Omitting it avoids accidental output.
- `//` and `#` comments end at a newline; `/* ... */` comments can span lines. Do not put executable code inside a comment and assume the parser will recover if delimiters are unbalanced.
- A string delimiter is part of the grammar. Single-quoted and double-quoted strings do not interpolate the same way; heredoc interpolates, while nowdoc behaves like a single-quoted block.
- Heredoc and nowdoc closing identifiers have strict placement rules in modern PHP. Keep the terminator visually obvious and run the project’s supported PHP version when checking it.
- A comma after the last item in many modern lists improves diffs, but trailing-comma support depends on the syntactic position and supported PHP version. Verify before using a form in a cross-version library.
- A namespace applies to the declarations in its namespace block. A file may use bracketed namespace blocks, but mixing styles carelessly makes name resolution difficult.
- `include`, `require`, and their `_once` variants are executable constructs. The source can parse successfully while the included file is missing or contains a parse error.
- A parse error in a file that is required during bootstrap can prevent all later error handling from loading. Lint entry points and dependencies in CI.

## Performance

Readable syntax is usually not a meaningful runtime bottleneck. The parser and compiler do work before execution, and OPcache can reuse compiled code in production. Performance decisions should therefore focus on behavior expressed by the syntax:

- a loop that performs a database query on each iteration;
- repeated string concatenation of a very large output;
- materializing a large array in a literal or transformation;
- including files repeatedly when an autoloader or OPcache configuration is misused;
- evaluating a regular expression or JSON transformation unnecessarily.

Changing `if` to a clever expression rarely solves a measured bottleneck. First identify whether the cost is PHP CPU, allocation, database work, I/O, or response transfer. Concise syntax can hide expensive work just as easily as verbose syntax can.

## Security

Syntax creates several security-sensitive boundaries:

- escape output for its destination context;
- never concatenate untrusted input into SQL, shell commands, or file paths;
- do not use `eval()` to turn input into PHP syntax;
- treat dynamic `include` paths as code-loading decisions, not ordinary string formatting;
- avoid logging secrets merely because string interpolation makes it easy;
- configure error display and error logging appropriately for development and production.

A value can be syntactically valid and still be an attack payload. The language parser checking that a string is a string does not make its contents safe.

## Database Interaction

Syntax does not determine whether data access is safe. This is vulnerable because the SQL string is assembled from input:

```php
$email = $_GET['email'] ?? '';
$sql = "SELECT id FROM users WHERE email = '$email'";
$row = $pdo->query($sql)->fetch();
```

The PHP expression is valid, but the database boundary is unsafe. Use a prepared statement and validate the input’s domain meaning:

```php
$email = trim((string) ($_GET['email'] ?? ''));

$statement = $pdo->prepare(
    'SELECT id FROM users WHERE email = :email',
);
$statement->execute(['email' => $email]);
$row = $statement->fetch();
```

Prepared statements address SQL syntax injection; they do not authenticate the caller, authorize access to the row, or prove that an email is valid for the application’s business rules.

## Testing

Syntax should be tested at several levels:

- Run `php -l` on changed PHP files using the project’s minimum supported PHP version.
- Run a formatter and static analyzer so malformed structure and inconsistent contracts are caught before review.
- Execute representative templates and CLI entry points, not only isolated functions.
- Test output as output: verify escaping, whitespace that is part of a protocol, and response headers where relevant.
- Test failure during bootstrap, such as a missing required file, in an integration environment where the application can report it safely.

A linter proves that a file parses. It does not prove that a branch is reachable, a query is authorized, or output is escaped. Pair syntax checks with behavior and boundary tests.

## Common Mistakes

- Treating whitespace outside PHP tags as harmless in a library file.
- Omitting braces around a conditional because the block currently has one statement.
- Using `=` when the intention was comparison.
- Relying on implicit operator precedence in a business rule.
- Assuming `use` loads a class or that a namespace changes runtime state.
- Embedding unescaped values in HTML because the template itself is trusted.
- Running new syntax on a newer local PHP than the deployment fleet supports.
- Calling `eval()` or dynamically including files to avoid designing a real parser or dispatch table.
- Treating a successful syntax check as evidence that the application is correct.

## Senior Engineer Thinking

When a syntax choice is under review, ask:

1. Which PHP versions must parse this file?
2. Where is the boundary between code and data or output?
3. Does this syntax make the value flow and failure path easier to inspect?
4. Is a short expression hiding a side effect or a precedence dependency?
5. Will a future edit preserve the same block boundaries?
6. Can the file be linted, tested, and loaded without unrelated global state?

Experienced engineers use syntax as a correctness tool. They prefer code that exposes ownership, boundaries, and assumptions over code that merely saves characters.

## Exercises

1. Write a PHP-only file that formats a court label and lint it with `php -l`. Add a deliberate missing brace, observe the parse error, and restore the file.
2. Convert a mixed PHP/HTML template so every value displayed in an HTML text context is escaped. Add a test value containing `<script>` and verify that it is rendered as text.
3. Rewrite a nested condition with explicit braces and parentheses. Explain which expressions produce values and which statements control execution.
4. Create a CLI script that accepts a positive integer and one of three surface names. Keep argument reading outside the validator and test missing, malformed, and valid arguments.
5. Find one place in an existing project where syntax hides a side effect—such as assignment in a condition, a function call inside interpolation, or an unbraced branch—and rewrite it for reviewability.

## Review Questions

1. What is the difference between a token, an expression, and a statement?
2. Why can a parse error not be handled by a `try`/`catch` inside the malformed file?
3. Why is omitting the closing PHP tag useful in PHP-only files?
4. What does a namespace or `use` statement change, and what does it not do?
5. Why is escaping output a context-specific operation?
6. How can valid syntax still produce a security vulnerability?
7. Why should the minimum supported PHP version be part of syntax decisions?
8. What kinds of performance costs can concise syntax hide?

## Summary

PHP syntax is the structure that turns source text into a program the runtime can compile and execute. PHP tags define code and output regions; semicolons terminate instructions; braces define blocks; expressions produce values; names and namespaces establish symbol identity. Parse-time validity is necessary but says nothing about runtime data, authorization, database safety, or output encoding. Keep PHP-only files free of accidental output, use explicit structure at important boundaries, lint against the supported version, and treat syntax as part of the application’s correctness and security design.

