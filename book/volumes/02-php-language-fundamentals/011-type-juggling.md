---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 11
title: Type Juggling
slug: type-juggling
status: complete
summary: ../../_ai/chapter-summaries/011-type-juggling-summary.md
---

# Chapter 11 — Type Juggling

## Why This Matters

PHP can interpret a value as another type when an operation requires it. This behavior—type juggling or coercion—is useful at boundaries, because form fields and many transport formats arrive as strings. It is also a frequent source of bugs when a string, integer, boolean, and `null` are treated as interchangeable.

The core distinction is:

```text
the value's current type  ≠  the type a context asks it to act like
```

An arithmetic expression, a conditional, a comparison, a function call, and a string interpolation can each use different conversion rules. “PHP converts values” is too vague to guide a review. Ask which context is performing the interpretation and whether that interpretation matches the application's contract.

## Mental Model

Type juggling is contextual:

```text
value + context → interpretation or failure
```

For example, the string `'42'` remains a string when stored in a variable, but it may be interpreted numerically in arithmetic or accepted by a coercive scalar parameter declaration. The value is not permanently transformed merely because one operation interpreted it.

```php
$raw = '42';

var_dump($raw);       // string(2) "42"
var_dump($raw + 1);   // int(43)
var_dump($raw);       // still string(2) "42"
```

The result of a conversion can be assigned to a new variable when a stable type is needed:

```php
$quantity = filter_var($raw, FILTER_VALIDATE_INT);
```

Validation and coercion are not the same. Coercion asks, “Can this value be interpreted as the requested type?” Validation asks, “Does this value satisfy the domain rule, and what should happen if it does not?”

## Core Concept

### Numeric contexts

Arithmetic operators require numeric interpretation:

```php
$count = '3';
$price = 12.50;

$total = $count * $price; // float 37.5
```

Numeric strings that represent integers or floating-point values can be interpreted accordingly. A non-numeric string should not be silently treated as a valid quantity. In current PHP versions, using a non-numeric string in a numeric operation can result in a `TypeError`; older code and older PHP versions may expose different warnings or conversions. Pin the behavior to the supported runtime rather than relying on folklore.

Check numeric input deliberately:

```php
$rawQuantity = $input['quantity'] ?? null;
$quantity = filter_var($rawQuantity, FILTER_VALIDATE_INT);

if ($quantity === false || $quantity < 1) {
    throw new InvalidArgumentException('Quantity must be a positive integer.');
}
```

`filter_var` returning `false` is different from returning integer `0`, so use an identity comparison when checking its result.

### Boolean contexts

Conditions interpret values as true or false. Some values are falsey, including `false`, `0`, `0.0`, the empty string, the string `'0'`, an empty array, and `null`:

```php
$input = '0';

if ($input) {
    echo 'present';
} else {
    echo 'falsey';
}
```

This does not answer whether a required field is present, valid, or authorized. A quantity of zero may be invalid, while a page offset of zero is perfectly valid. Use a domain-specific check instead of treating falsey as synonymous with missing:

```php
if ($input === null || $input === '') {
    throw new InvalidArgumentException('Value is required.');
}
```

### String contexts

Values can be interpreted as strings in interpolation, concatenation, and string declarations. Objects may define `__toString()`, while arrays and some other values cannot be converted meaningfully:

```php
$id = 42;
$message = 'Reservation #' . $id;
```

String conversion is not escaping. A string safe for a log line may be unsafe for HTML, a URL, SQL, or a shell command. Type juggling solves representation, not destination security.

### Comparison contexts

Loose equality (`==`, `!=`) can compare values after type juggling. Strict identity (`===`, `!==`) requires matching value and type:

```php
var_dump(0 == '0');   // true
var_dump(0 === '0');  // false
```

Use strict comparisons when the type is part of the meaning. This is especially important for authentication and authorization decisions, where a loosely equal but differently typed value can produce an unintended branch.

The comparison rules have changed across PHP versions, particularly for number-to-string comparisons. Do not build new business rules around a memorized comparison table. Normalize values and compare the normalized representation with strict operators.

### Function contexts

Without `declare(strict_types=1)`, user-defined scalar parameter and return declarations generally allow limited coercion. With strict types enabled in the caller's file, a mismatched scalar argument normally produces a `TypeError`, with the documented exception that an integer can satisfy a `float` declaration.

```php
function addOne(int $value): int
{
    return $value + 1;
}

addOne('41'); // Coercive or failing behavior depends on the calling file.
```

The strictness setting is per file and applies to calls made from that file; it is not a global switch for the entire application. Chapter 12 covers this in detail. The practical rule for this chapter is to know whether a boundary is intentionally coercive, and to normalize external values before core logic relies on them.

### Union types have their own coercion rules

In coercive mode, a union such as `int|float|string` can accept values through limited conversion. The exact selected type depends on the declared alternatives and the value, including numeric-string semantics:

```php
function show(int|float|string $value): string
{
    return get_debug_type($value) . ': ' . (string) $value;
}
```

Do not use a broad union to avoid deciding what the application means. `int|string` may be correct for an identifier that genuinely has both numeric and opaque forms; it is a poor replacement for normalizing inconsistent request data.

## Minimal Example

```php
<?php

declare(strict_types=1);

$rawQuantity = '3';

if (!is_numeric($rawQuantity)) {
    throw new InvalidArgumentException('Quantity must be numeric.');
}

$quantity = (int) $rawQuantity;

if ((string) $quantity !== $rawQuantity) {
    throw new InvalidArgumentException('Quantity must be an integer string.');
}

echo $quantity * 1250, PHP_EOL;
```

This example makes conversion explicit and then applies an additional domain check. `is_numeric` alone would accept numeric forms that may not be appropriate for a positive integer quantity, so the application still needs to define its grammar.

## Practical Example

Normalize identifiers at the boundary instead of comparing raw alternatives throughout the application:

```php
function parseUserId(mixed $raw): int
{
    if (is_int($raw)) {
        $userId = $raw;
    } elseif (is_string($raw) && preg_match('/^[1-9][0-9]*$/', $raw) === 1) {
        $userId = (int) $raw;
    } else {
        throw new InvalidArgumentException('Invalid user id.');
    }

    if ($userId < 1) {
        throw new InvalidArgumentException('Invalid user id.');
    }

    return $userId;
}
```

After this function returns, downstream code can use `int $userId`. The regular expression expresses the accepted wire format, while the range check expresses the domain rule. An unchecked cast such as `(int) $raw` would turn many invalid strings into `0`, hiding an input error.

## Bad Example

Loose comparison is dangerous at a credential boundary:

```php
if ($providedToken == $storedToken) {
    grantAccess();
}
```

The comparison may interpret values rather than requiring the exact string representation. Use an explicit string contract and a timing-resistant comparison for secrets:

```php
if (is_string($providedToken)
    && is_string($storedToken)
    && hash_equals($storedToken, $providedToken)
) {
    grantAccess();
}
```

This still does not authenticate a user by itself; it only makes this comparison's type and timing behavior appropriate for a token check.

## Better Example

Keep coercion at one adapter boundary and pass a domain value inward:

```php
final readonly class Quantity
{
    public function __construct(public int $value)
    {
        if ($value < 1) {
            throw new InvalidArgumentException('Quantity must be positive.');
        }
    }
}

function createLineItem(Quantity $quantity): void
{
    // No string-to-number interpretation is required here.
}
```

The constructor is not a replacement for authorization or inventory checks. It simply ensures that every `Quantity` object has the same local invariant.

## Edge Cases

### `null` is not the same as an empty string

Request fields, decoded JSON, database drivers, and environment variables can represent absence differently. Do not normalize every absent value to `''` or `0` before deciding what absence means.

### Boolean strings are surprising

The string `'false'` is non-empty and therefore truthy. It does not become boolean `false` merely because its contents spell the word “false”:

```php
var_dump((bool) 'false'); // true
```

Parse protocol booleans explicitly, accepting only the representations the protocol defines.

### Numeric strings have grammar and range

Whitespace, signs, decimal points, exponent notation, leading zeroes, and values outside the integer range can change interpretation. If an API requires a positive decimal integer, define that grammar rather than accepting every value that PHP can treat as numeric.

### Floating-point equality

Two calculations that appear mathematically equal may differ in binary floating-point representation. Compare within a domain-appropriate tolerance or use integer minor units/decimal arithmetic when exactness matters.

## Performance

Implicit conversion is rarely the main cost in a request. The larger risks are repeated parsing, repeated normalization, and allowing ambiguous values to travel through many layers. Normalize once when the boundary is stable and reuse the typed result.

Explicit validation also has a maintenance performance benefit: it reduces repeated branches and makes static analysis more effective. Do not add elaborate conversion machinery to a hot path without measuring; choose the simplest grammar that protects the actual contract.

## Security

Type juggling is security-sensitive wherever a comparison or branch controls access, money, file selection, command execution, or message handling. Use strict comparisons after normalization, parameterize SQL, escape output for its context, and never treat a truthy value as proof of authentication or authorization.

For secrets, compare strings with `hash_equals` after validating that both operands are strings. For passwords, use password hashing APIs rather than comparing or casting values manually.

## Database Interaction

Database drivers may return scalar values in transport-specific types. Decide whether a repository returns normalized domain values or exposes driver representations. Do not let a string `'1'` versus integer `1` distinction leak into authorization logic accidentally.

Prepared statements prevent SQL syntax injection, but they do not determine whether a value is the correct domain type or whether the caller may access the row. Type normalization, parameterization, and authorization solve different problems.

## Testing

Test the contexts that matter, not only casts in isolation:

```php
public function testUserIdParserAcceptsWireString(): void
{
    self::assertSame(42, parseUserId('42'));
}

public function testUserIdParserRejectsZero(): void
{
    $this->expectException(InvalidArgumentException::class);

    parseUserId('0');
}

public function testFalseStringIsNotBooleanFalse(): void
{
    self::assertTrue((bool) 'false');
}
```

Include tests for the minimum and maximum accepted values, malformed numeric strings, explicit nulls, empty strings, boolean spellings, loose versus strict comparisons, and the actual transport decoder used by the application.

## Common Mistakes

- Treating coercion as validation.
- Using `==` for identity, credentials, or authorization decisions.
- Assuming the string `'false'` is false.
- Casting malformed input to `int` and accepting the resulting zero.
- Treating `is_numeric` as proof of an integer domain value.
- Forgetting that strictness is configured per calling file.
- Using `float` when exact monetary arithmetic is required.
- Assuming a string's type makes its contents safe for a destination.
- Letting database driver types dictate domain semantics accidentally.

## Senior Engineer Thinking

When reviewing a conversion, ask:

1. Which context is requesting the interpretation?
2. Is the operation coercion, validation, normalization, or comparison?
3. What invalid values become plausible valid defaults after conversion?
4. Does this decision control security, money, identity, or data selection?
5. Would normalizing once at the boundary simplify every downstream caller?
6. Which PHP version and calling-file strictness control the behavior?
7. Is a cast hiding an input contract that should be named and tested?

The mature response to type juggling is not fear of every conversion. It is making each conversion intentional, local, testable, and appropriate to its context.

## Exercises

1. Build a parser for a positive integer query parameter. Define accepted strings and reject whitespace, signs, decimals, empty strings, and overflow.
2. Write tests comparing `0`, `'0'`, `false`, `null`, `''`, and `'false'` with both loose and strict operators. Explain which comparisons fit a real domain rule.
3. Take a token comparison that uses `==` and replace it with explicit string validation and `hash_equals`.
4. Create two callers—one strict and one coercive—for the same scalar-typed function. Document where the strictness decision is made.
5. Identify a database field whose driver type differs from its domain type. Add a mapping step and tests for both representations.

## Review Questions

1. What does it mean for type juggling to be contextual?
2. Why is coercion not the same as validation?
3. Why can a falsey value still be valid input?
4. Why is `'false'` truthy in a boolean context?
5. When should `===` be preferred to `==`?
6. Where is the strictness decision made for a user-defined function call?
7. Why can `(int) $rawInput` hide malformed input?
8. Why are numeric-string grammar and range part of an API contract?
9. Why does type normalization not solve authorization or injection?

## Summary

PHP interprets values according to context: numeric operations, boolean conditions, string operations, comparisons, and function calls do not all use the same rules. A value can be interpreted without changing the original variable's type. Coercion is not validation, and a cast is not a domain contract.

Normalize external values deliberately, use strict comparisons for identity and security decisions, define numeric and boolean grammars explicitly, and test the actual boundary and PHP version. Chapter 12 builds on this foundation to explain strict scalar declarations and their per-file calling semantics.

## Sources and Further Reading

- [PHP Manual: Type Juggling](https://www.php.net/types.type-juggling)
- [PHP Manual: Type declarations](https://www.php.net/types.declarations)
- [PHP Manual: Numeric strings](https://www.php.net/manual/en/language.types.numeric-strings.php)
- [PHP Manual: Comparison tables](https://www.php.net/manual/en/types.comparisons.php)
