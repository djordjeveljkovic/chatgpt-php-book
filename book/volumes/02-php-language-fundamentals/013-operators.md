---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 13
title: Operators
slug: operators
status: complete
summary: ../../_ai/chapter-summaries/013-operators-summary.md
---

# Chapter 13 — Operators

## Why This Matters

Operators look small on the page, but they decide how values combine, compare, branch, mutate, and flow through a program. A one-character difference—`==` versus `===`, `&&` versus `and`, `+` versus `.`, `??` versus `?:`—can change a security decision, a query condition, a price, or whether a side effect runs.

The reliable habit is to read an operator expression as a small program: identify the operands, determine their types, establish grouping with parentheses, and check whether evaluation can short-circuit or mutate state. Operators are executable design decisions, not punctuation to skim past.

## Mental Model

An operator takes one or more operands and produces a value. Unary operators take one operand (`!$enabled`), binary operators take two (`$a + $b`), and the conditional operator takes three (`$ok ? $yes : $no`). Assignment also produces a value, which is why chained assignments are legal.

Separate three questions:

1. How is the expression grouped? Precedence and associativity answer this.
2. Which operands are actually evaluated? Short-circuit and conditional behavior answer this.
3. What conversions or side effects occur? The operator and operand types answer this.

```php
$total = ($unitPriceCents * $quantity) + $shippingCents;
```

Parentheses make intended grouping visible even where PHP's precedence would produce the same result. They are executable documentation.

## Core Concept

### Arithmetic and numeric results

PHP provides `+`, `-`, `*`, `/`, `%`, and `**`. Division with `/` produces a floating-point result when appropriate; integer division can be expressed with `intdiv()`. Division by zero throws `DivisionByZeroError` for ordinary division and integer division. `fdiv()` is useful when a floating-point result for a zero divisor is intentionally part of the contract, but it is not a way to hide invalid input.

```php
$subtotal = 1250 * 3;
$average = $subtotal / 2;
$wholeBatches = intdiv(10, 3);
$remainder = 10 % 3;
```

Money calculations should not rely on binary floating-point arithmetic. Store currency as integer minor units or use a decimal strategy with explicit rounding. An operator can be mathematically correct while the representation is unsuitable for accounting.

### Assignment and compound assignment

`=` stores a value and itself evaluates to that value:

```php
$a = $b = 3; // $b becomes 3, then $a receives 3
```

Compound operators such as `+=`, `-=`, `*=`, `/=`, `.=` and `??=` read the left side, apply an operation, and store the result:

```php
$attempts = 0;
$attempts += 1;
$label = 'court';
$label .= ' A';
$displayName ??= 'Guest';
```

Mutation is useful when it is the point. Avoid hiding meaningful state changes inside conditions or arguments.

### Comparison: equality, identity, and ordering

Use `===` and `!==` when both type and value matter. Use `==` only when conversion is part of the explicit contract. `<`, `<=`, `>`, and `>=` order values according to PHP's comparison rules; `<=>` returns `-1`, `0`, or `1` and is useful for comparator functions.

```php
if ($status === 'confirmed') {
    allowNextStep();
}

$direction = $left <=> $right;
```

The spaceship operator does not solve an ill-defined ordering. Decide how nulls, strings, case, and invalid values should rank before writing a comparator. Do not compare floats for exact equality; use a domain-appropriate tolerance or an exact representation.

### Logical operators and short-circuiting

`&&` and `||` short-circuit: the right operand is evaluated only when necessary. `!` negates a boolean interpretation.

```php
if ($user !== null && $user->isActive()) {
    allowAccess($user);
}
```

Do not put an important side effect in a short-circuited operand:

```php
$authorized && auditAccess($user); // audit may never happen
```

The word operators `and`, `or`, and `xor` have different precedence from `&&` and `||`:

```php
$result = true and false; // ($result = true) and false; $result is true
$result = true && false;  // $result is false
```

Choose one style consistently. In ordinary conditions, `&&`, `||`, and `!` are easier to scan; use parentheses when grouping matters.

### Null coalescing and ternaries

`$value ?? $fallback` returns the left value when it exists and is not `null`; otherwise it returns the fallback. It is useful for optional input, but it does not validate the value:

```php
$page = $_GET['page'] ?? 1; // page may still be a malformed string
```

The shorthand ternary `$value ?: $fallback` uses truthiness, so it treats `0`, `'0'`, `false`, `''`, `null`, and an empty array as absent. Use `??` for null/missing semantics and `?:` only when false-like values are intentionally equivalent.

```php
$limit = $requestedLimit ?? 25;
$title = $providedTitle ?: 'Untitled';
```

The full ternary chooses one of two expressions. Since PHP 8.0, chaining ternaries without parentheses is not allowed; write the grouping explicitly or use `match`.

### String and array operators

`.` concatenates strings and `.=` appends to a string. Since PHP 8.0, `.` has lower precedence than arithmetic `+` and `-`, so mixed expressions should be parenthesized:

```php
$message = 'Total: ' . ($subtotal + $shipping);
```

The array union operator `+` preserves keys from its left operand and adds keys missing from the left. It is not list concatenation:

```php
$defaults = ['timeout' => 5, 'retries' => 2];
$custom = ['timeout' => 10];

$config = $custom + $defaults; // custom timeout wins
$list = [1, 2] + [3, 4];       // [1, 2], because keys 0 and 1 exist
```

Array equality (`==`) compares key/value pairs after loose comparison rules; identity (`===`) also requires the same order and types. Use explicit merge functions when the policy is not obvious.

### Bitwise and type operators

`&`, `|`, `^`, `~`, `<<`, and `>>` operate on integer bits, and bitwise operators also have documented string behavior. They are appropriate for masks, compact flags, and low-level formats—not as a substitute for ordinary boolean logic.

`instanceof` tests whether an object is an instance of a class, interface, or compatible type. `?->` performs null-safe method or property access and returns `null` when the receiver is null. It does not catch arbitrary exceptions or make a missing domain object valid.

### The PHP 8.5 pipe operator

PHP 8.5 introduced `|>`. It passes the value on the left as the single argument to a callable on the right and evaluates left to right:

```php
$slug = ' PHP 8.5 Released '
    |> trim(...)
    |> (fn (string $value): string => str_replace(' ', '-', $value))
    |> strtolower(...);
```

The pipe is useful for pure transformations. It does not make calls asynchronous, remove the need for error handling, or eliminate their runtime cost. The callable must accept the piped value as its argument; arrow functions in a pipe chain need parentheses.

## How It Works

For an expression with several operators, precedence determines grouping and associativity determines grouping among equal-precedence operators. Neither guarantees the order in which unrelated subexpressions with side effects are evaluated.

```php
$value = 2 + 3 * 4;       // 14: multiplication groups first
$value = (2 + 3) * 4;     // 20: parentheses change the grouping
$value = $a = $b = 1;     // right-associative assignment
```

A simplified reasoning process is:

```text
parse expression → establish grouping → evaluate required operands → apply operation → produce result
```

The parser and compiler turn the expression into executable instructions; the Zend VM performs the operation at runtime. Precedence is a language rule, while cost also depends on operand types, allocations, and external calls inside operands.

## What PHP Does

PHP evaluates expressions in assignments, function arguments, conditions, return expressions, array keys, and interpolation. Operators do not automatically make a value safe. `===` prevents a type conversion in that comparison, but does not sanitize a string; `??` supplies a fallback, but does not parse a query parameter; `+` adds values, but does not establish that they represent money.

Ask both “what value does this produce?” and “what contract does that value represent?”

## Minimal Example

```php
<?php

declare(strict_types=1);

$priceCents = 1250;
$quantity = 3;
$shippingCents = 500;

$totalCents = ($priceCents * $quantity) + $shippingCents;

var_dump($totalCents);           // int(4250)
var_dump($totalCents === 4250);  // bool(true)
var_dump($totalCents == '4250'); // bool(true), loose comparison
```

The calculation uses integers and explicit grouping. The final line illustrates a language rule, not a recommendation for a domain invariant.

## Practical Example

An availability check combines comparisons and logical operators. Half-open intervals `[start, end)` treat an appointment ending at 10:00 as non-overlapping with one beginning at 10:00:

```php
final readonly class TimeRange
{
    public function __construct(
        public int $startsAt,
        public int $endsAt,
    ) {
        if ($startsAt >= $endsAt) {
            throw new InvalidArgumentException('A range must have positive length.');
        }
    }

    public function overlaps(self $other): bool
    {
        return $this->startsAt < $other->endsAt
            && $other->startsAt < $this->endsAt;
    }
}
```

The two `<` comparisons encode the interval policy; `&&` short-circuits if the first comparison is false. The expression is O(1) for one pair. Checking `n` existing reservations remains O(n) unless the data structure or database query changes.

## Production Example

A filtering pipeline should expose steps that can be tested separately:

```php
function normalizeTitle(string $title): string
{
    return $title
        |> trim(...)
        |> (fn (string $value): string => preg_replace('/\\s+/', ' ', $value) ?? $value)
        |> strtolower(...);
}
```

This is readable when each step is a pure, cheap transformation. If a step performs I/O, hides a database query, or can fail with a business decision, named statements are clearer:

```php
$title = trim($rawTitle);
$title = collapseWhitespace($title);
$title = normalizeCase($title);
```

The operator is not the architecture. Data flow, error policy, and cost still need to be visible.

## Bad Example

```php
if (!$user = findUser($id) || !$user->isActive()) {
    deny();
}
```

This expression mixes assignment, negation, `||`, method access, and a side-effecting lookup. Its grouping is easy to misread, and the variable may not contain what the author expects.

Another common failure is:

```php
echo 'Total: ' . $subtotal - $discount;
```

The grouping is not the intended “total label plus arithmetic result.”

## Better Example

```php
$user = findUser($id);

if ($user === null || !$user->isActive()) {
    deny();
}
```

And:

```php
echo 'Total: ' . ($subtotal - $discount);
```

The improved code gives each operation a name or explicit grouping. It is easier to test, instrument, and modify.

## Edge Cases

- Pre-increment (`++$i`) changes the value before the expression receives it; post-increment (`$i++`) changes it afterward. Do not combine increments with other reads of the same variable in one expression.
- PHP does not promise a useful order for unrelated side-effecting subexpressions. Split them into statements.
- `and`/`or` have lower precedence than assignment; `&&`/`||` do not. Never rely on readers remembering this from a clever one-liner.
- `??` handles an undefined left variable/index but does not reject an existing null or malformed value.
- `?:` uses truthiness, so a legitimate zero can unexpectedly select the fallback.
- `==` and ordering comparisons have type-dependent rules. For security and sentinel checks, prefer identity.
- The ternary operator is non-associative in modern PHP; nested ternaries need parentheses.
- The error-control `@` operator suppresses diagnostics and can conceal operational failures. It is not error handling.
- Backticks execute a shell command and are deprecated in PHP 8.5; use an explicit process API with argument handling and a security review when shell execution is genuinely required.

## Performance

Arithmetic and boolean operations are cheap relative to I/O, but expression shape still matters. Short-circuiting can avoid an expensive function call, while a side-effecting condition can make behavior hard to cache or reason about. A pipe chain expresses sequential calls; it does not make them asynchronous or eliminate their allocations and runtime cost.

For collections, remember that `+` is array union, not a general merge, and that materializing intermediate arrays has memory costs. Choose a database query, generator, or streaming design when the dataset makes PHP-side arrays expensive.

## Security

Use strict comparison for authentication states, permission flags, method names, and security tokens. Avoid loose comparisons involving user input. Be careful with `||` and `&&` around authorization: short-circuiting can mean an audit or policy check never runs if it is placed on the wrong side.

Do not use backticks or shell execution with interpolated input. If a process is genuinely required, pass fixed arguments through a well-defined boundary, validate input, constrain privileges, and capture failures. `@` can hide warnings that reveal a security or availability problem.

## Database Interaction

Operators in PHP should not imitate database guarantees. This is insufficient for a reservation:

```php
if (canReserve($request, $existing)) {
    insertReservation($request);
}
```

The comparison operators may correctly detect overlap in the snapshot the process read, while another transaction inserts a conflicting row before this one commits. Use a transaction, appropriate isolation or locking, and database constraints where the schema can express the invariant. PHP operators express local logic; they do not serialize concurrent writers.

Likewise, filtering a million rows with `array_filter()` after fetching them is usually inferior to a parameterized indexed query. Ask which side of the boundary owns filtering, sorting, aggregation, and uniqueness.

## Concurrency

Short-circuiting is local control flow, not concurrency control. `??=` is not atomic distributed initialization, and `$counter++` is not a safe shared counter across PHP-FPM workers. Each worker has its own process memory; shared state needs a database, Redis, a queue, or another system with an explicit atomicity model.

At a shared-state boundary, design the operation as a transaction or atomic command. Then test races and retries rather than assuming a correct operator expression makes the workflow safe.

## Testing

Test grouping and boundary values directly:

```php
final class OperatorRulesTest extends TestCase
{
    public function testArithmeticIsGroupedBeforeConcatenation(): void
    {
        self::assertSame('Total: 9', 'Total: ' . (5 + 4));
    }

    public function testZeroIsNotMissingWhenUsingNullCoalescing(): void
    {
        $page = 0;
        self::assertSame(0, $page ?? 1);
    }

    public function testZeroIsFalseLikeForElvisTernary(): void
    {
        self::assertSame(1, 0 ?: 1);
    }

    public function testAdjacentRangesDoNotOverlap(): void
    {
        $first = new TimeRange(9, 10);
        $second = new TimeRange(10, 11);

        self::assertFalse($first->overlaps($second));
    }
}
```

Add tests for both branches of short-circuit conditions without relying on a hidden side effect to prove evaluation. For a comparator, test equal, less-than, greater-than, null, and malformed inputs according to the domain policy. If adopting the PHP 8.5 pipe operator, include a minimum-version check in CI and a readable equivalent for supported older runtimes.

## Common Mistakes

- Using `==` for identifiers, permissions, or tokens.
- Mixing arithmetic and concatenation without parentheses.
- Writing `$ok = condition() and record()` and assuming the assignment includes both operations.
- Placing a required side effect in a short-circuited operand.
- Using `?:` when zero or an empty string is a valid value.
- Treating `??=` as safe shared-state initialization.
- Calling the database after a PHP-only availability check without handling races.
- Building clever pipelines that hide I/O, exceptions, or expensive work.

## Senior Engineer Thinking

For a non-trivial expression, ask:

1. What are the operand types at runtime?
2. What grouping does PHP parse, and would a future reader see the same grouping?
3. Which operands can be skipped, and is skipping safe?
4. Does the operator compare representations or domain identity?
5. Is an assignment or mutation hidden inside the expression?
6. Does this operation belong in PHP, the database, or a shared atomic system?
7. What happens on retry, under concurrency, or when an operand throws?

The senior move is often to split one expression into three named statements. Fewer characters do not necessarily mean less complexity.

## Exercises

1. Predict and then run expressions mixing `=`, `and`, `&&`, `??`, `?:`, `.`, `+`, and `-`. Rewrite each with parentheses and named variables.
2. Implement a comparator for reservations ordered by start time, then court ID, then creation ID. Define equal values and test them.
3. Write a pure pipe chain for normalizing a title in PHP 8.5, then rewrite it with intermediate variables. Compare readability, debugging, and minimum runtime version.
4. Review a reservation creation flow and mark every operator expression that could be wrong under concurrent requests. Propose the database operation that must replace the local check.

## Review Questions

1. What is the difference between precedence, associativity, and evaluation order?
2. Why is `===` usually safer than `==`?
3. How do `??` and `?:` differ when the left value is `0` or an empty string?
4. Why are `and` and `&&` not interchangeable in assignments?
5. What does short-circuiting change, and why should side effects be kept out of it?
6. Why is PHP array union different from list concatenation or a general merge?
7. What does the PHP 8.5 pipe operator do, and when would named statements be better?
8. Why can a correct local comparison still fail to enforce a database invariant?

## Summary

Operators are small executable programs: they group operands, decide which expressions run, apply type-sensitive rules, and return values that may mutate state. Use parentheses and named intermediates, prefer identity comparisons, distinguish `??` from truthiness-based `?:`, keep side effects out of short-circuited expressions, and treat PHP 8.5's pipe as a readability tool rather than an architectural shortcut. Local operators cannot replace database constraints or concurrency control.

### Further reading

- [PHP Manual: Operators](https://www.php.net/manual/en/language.operators.php)
- [PHP Manual: Operator precedence](https://www.php.net/manual/en/language.operators.precedence.php)
- [PHP Manual: Comparison operators](https://www.php.net/manual/en/language.operators.comparison.php)
- [PHP 8.5 release announcement](https://www.php.net/releases/8.5/en.php)
