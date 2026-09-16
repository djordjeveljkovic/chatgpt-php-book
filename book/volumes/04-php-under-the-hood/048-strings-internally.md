---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 48
title: Strings Internally
slug: strings-internally
status: complete
summary: ../../_ai/chapter-summaries/048-strings-internally-summary.md
---

# Chapter 48 — Strings Internally

## Why This Matters

In PHP, a string is a language value that can contain text, serialized data, a protocol payload, or arbitrary bytes. Internally, Zend must store its length, contents, ownership state, and often a hash value. Those choices explain why PHP strings are binary-safe, why string length is not character count, why concatenation can allocate, and why using a string as a HashTable key can be cheaper than repeatedly hashing the same contents.

The important distinction is:

```text
PHP string semantics       → what application code may rely on
zend_string representation  → current engine implementation detail
encoding/locale policy      → application and extension responsibility
```

Do not infer Unicode behavior from the internal name `zend_string`. Zend stores bytes; an encoding-aware operation requires the appropriate PHP function or extension.

## Mental Model

```text
zval(type = string)
        │
        ▼
zend_string
├── refcount / GC metadata
├── cached hash field
├── byte length
└── NUL-terminated byte buffer
```

The final NUL makes interoperability with C routines possible, but the explicit length is authoritative for PHP strings. A NUL byte in the middle is data, not the end of the PHP value.

## Core Concept

Current php-src represents a string with a `zend_string` structure containing a refcounted header, a hash field, a length, and a flexible character buffer. The exact field ordering, header flags, allocator behavior, and macro definitions are version-sensitive. The model is stable enough to explain behavior; it is not a portable C ABI specification.

```c
/* Conceptual pseudocode, intentionally not an ABI definition. */
struct zend_string {
    refcounted_header gc;
    unsigned long hash;
    size_t length;
    char value[length + 1];
};
```

The engine can therefore distinguish these values correctly:

```php
$binary = "a\0b";

var_dump(strlen($binary)); // 3
var_dump($binary === "a\0b"); // true
```

The C buffer’s terminating NUL does not remove the embedded NUL from PHP’s length-counted value.

## How It Works

### Creation and ownership

A string literal is compiled into runtime data and may be represented as a shared or interned string depending on the build and lifecycle. A dynamically created string is allocated and managed by Zend’s memory and refcounting machinery:

```php
$prefix = 'report:';
$name = 'january.csv';
$path = $prefix . $name;
```

Conceptually:

```text
$prefix ──► string("report:")
$name   ──► string("january.csv")
$path   ──► newly produced string("report:january.csv")
```

The engine may optimize particular literals or operations, but concatenation that changes the content generally needs a result buffer large enough for the resulting bytes. Repeated growth in a loop can therefore create allocation and copying pressure.

### Copy-on-write for strings

Strings are refcounted payloads and can be shared while read-only:

```php
$a = str_repeat('x', 1_000_000);
$b = $a;

$b[0] = 'y';

var_dump($a[0]); // x
var_dump($b[0]); // y
```

Before the offset write, both zvals can point to one payload. The write must preserve `$a`, so Zend separates the value if it is shared. This is an implementation strategy supporting PHP’s value semantics; code should not assume that every string assignment shares or that every mutation has the same allocation path.

### Hashes and interned strings

The `zend_string` structure includes a field used for a cached hash. For a string used repeatedly as a hash key, avoiding repeated hash computation can matter. The exact hash function and cache flags are implementation details; do not implement application security decisions from them.

Interned strings are strings stored in a table so equal text can be reused, especially for identifiers and other frequently repeated names. Interning can reduce duplicate storage and make identity checks useful internally. Interned-string lifetime and persistence depend on the string category, process/configuration, and engine version. It is not a userland promise that two equal PHP strings have the same address.

### Bytes versus characters

```php
$word = 'Ž';

echo strlen($word), PHP_EOL; // bytes in the current encoding, commonly 2 for UTF-8
echo mb_strlen($word, 'UTF-8'), PHP_EOL; // characters, with mbstring enabled
```

`strlen()` reports the number of bytes in the PHP string. `mb_strlen()` interprets a selected multibyte encoding. Grapheme clusters, normalization, and user-visible characters are still different questions; for those, use an appropriate Unicode-aware operation. The engine’s byte storage does not decide what a user considers one character.

### Interpolation and formatting

These forms have different parsing and runtime work:

```php
$message = "Hello, $name";
$message = 'Hello, ' . $name;
$message = sprintf('Hello, %s', $name);
```

They can all produce the same bytes for a given input, but they exercise different compiler and function paths. Choose for semantics and clarity first. Benchmark only a hot path with realistic values, because formatting cost may be dominated by logging I/O or downstream transport.

### String offsets

PHP supports offset access with byte-oriented semantics:

```php
$value = 'cat';
$value[0] = 'b';
echo $value; // bat
```

An offset is not a Unicode code-point operation. Writing a character that does not fit the expected byte-level operation can produce warnings/errors or different results across language versions; validate and test boundary cases rather than treating it as general text editing.

## What PHP Does

The PHP manual defines strings as sequences of bytes and documents quoting, interpolation, access, conversion, and binary safety in the [Strings](https://www.php.net/manual/en/language.types.string.php) section. It does not expose `zend_string` fields or guarantee internal interning.

For external data, make the encoding contract explicit:

```php
function userDisplayName(string $utf8): string
{
    if (!mb_check_encoding($utf8, 'UTF-8')) {
        throw new InvalidArgumentException('Expected UTF-8 input');
    }

    return $utf8;
}
```

This validates a policy at the application boundary. It does not change the underlying byte storage.

## What Zend Does

The PHP 8.4 implementation is documented by source such as [Zend/zend_string.h](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_string.h) and [Zend/zend_string.c](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_string.c). Those files define constructors, allocation/release helpers, interning behavior, hash handling, and string utility operations for that source branch.

The `zend_string` layout, hash implementation, string interning table, allocation paths, and optimization choices can change. Treat source links as a snapshot for learning and debugging, not as a stable application-level contract. Extensions should use the supported Zend API macros/functions for the target version.

## Minimal Example

```php
<?php

$payload = "header\0body";

printf("bytes=%d\n", strlen($payload));
printf("hex=%s\n", bin2hex($payload));
```

Expected conceptual output:

```text
bytes=11
hex=68656164657200626f6479
```

This is a useful protocol test: PHP preserves the NUL byte because the runtime tracks the explicit length.

## Practical Example: Building a Large Export

This pattern creates a new large result repeatedly:

```php
$csv = "id,state\n";
foreach ($rows as $row) {
    $csv .= $row['id'] . ',' . $row['state'] . "\n";
}
```

It may be perfectly appropriate for a small export, but it materializes the complete result in memory. For a large or streamed response, write chunks to an output stream instead:

```php
$handle = fopen('php://output', 'wb');
fwrite($handle, "id,state\n");

foreach ($rows as $row) {
    $line = $row['id'] . ',' . $row['state'] . "\n";
    fwrite($handle, $line);
}

fclose($handle);
```

The right choice depends on transport, buffering, error handling, and whether the caller needs a retryable complete value. The internal string model explains why “one giant string” has a memory cost; it does not by itself choose the API design.

## Bad Example

```php
// Treating a UTF-8 string as an array of characters.
$firstCharacter = $name[0];
```

For multibyte text, this can return only the first byte of a code point. It is correct only when the contract is explicitly byte-oriented, such as parsing a binary format or ASCII-compatible protocol.

## Better Example

```php
function firstCodePoint(string $text): string
{
    if ($text === '' || !mb_check_encoding($text, 'UTF-8')) {
        throw new InvalidArgumentException('Expected non-empty UTF-8');
    }

    return mb_substr($text, 0, 1, 'UTF-8');
}
```

For user-visible text, specify encoding and test malformed input. For binary data, keep byte operations and avoid silently passing it through Unicode transformations.

## Edge Cases

- An empty string has length zero but is still a valid string value.
- NUL bytes are data to PHP, even though many C APIs treat NUL as a terminator.
- `strlen()` counts bytes; it does not count Unicode characters or grapheme clusters.
- Concatenation, interpolation, formatting, and repeated mutation can take different allocation paths.
- Interning is an engine optimization/lifetime mechanism, not a stable address or identity guarantee for userland strings.
- Hash values are not a password hash and must not be exposed as a security primitive.
- Locale, encoding, normalization, and case-folding are distinct concerns from byte storage.
- A string used as an array key participates in key conversion and HashTable rules (Chapters 49–50).

## Performance

String work is approximately:

```text
cost ≈ bytes read + bytes copied/allocated + encoding work + I/O
```

Repeatedly constructing an O(n)-sized result can lead to substantial total copying, although the exact behavior depends on the operation and engine version. Prefer streaming or chunked writes when the consumer supports them. Avoid premature “micro-optimizations” such as relying on interning; measure allocations, peak memory, and end-to-end latency.

When benchmarking, vary input size. A method that is indistinguishable for a 100-byte label can be materially different for a 100 MB payload. Include malformed and multibyte inputs if validation or encoding conversion is part of the workload.

## Security

Strings frequently carry secrets, tokens, SQL, HTML, shell arguments, and file paths. Binary-safe storage does not make those contexts safe:

- use prepared statements for SQL;
- escape for the actual HTML context;
- validate and constrain paths;
- avoid shell interpretation or use safe process APIs;
- do not log credentials or raw authorization headers.

Be especially cautious with embedded NUL bytes at extension and operating-system boundaries. A PHP string can contain bytes that a downstream C API interprets differently. Validate the contract before crossing that boundary.

## Testing

Test strings by contract:

1. Include empty, ASCII, multibyte, malformed, and NUL-containing values.
2. Assert byte length separately from character length.
3. Test normalization/case rules if user-visible comparisons matter.
4. Test large payload behavior with peak-memory assertions or a process-level budget.
5. Test exact bytes at binary/protocol boundaries using `bin2hex()` or byte comparisons.

For version-sensitive behavior such as offset writes or deprecations, run the fixture under every supported PHP version and record the expected diagnostic policy.

## Common Mistakes

- Calling `strlen()` a character counter.
- Assuming PHP strings are NUL-terminated in the semantic sense.
- Treating `zend_string` addresses or interned status as application identity.
- Building huge responses in memory when the protocol supports streaming.
- Using string concatenation to create SQL, HTML, or shell commands without context-specific protection.
- Assuming equal bytes imply equal Unicode meaning without an encoding/normalization policy.

## Senior Engineer Thinking

Start with the string’s contract:

```text
data kind: text / binary / identifier / secret / protocol frame
    → encoding and length rules
    → mutation and ownership needs
    → allocation/streaming strategy
    → external boundary and escaping policy
    → tests for bytes, characters, errors, and scale
```

The internal representation is valuable because it predicts where bytes may be copied and why NUL/length behavior matters. The application contract remains the decision point.

## Exercises

1. Write tests comparing `strlen()` and `mb_strlen()` for ASCII and UTF-8 input.
2. Construct a string containing an embedded NUL and verify its bytes after a round trip through a file or binary-safe transport.
3. Benchmark a complete in-memory CSV export against chunked output at three input sizes.
4. Locate `zend_string` in a pinned php-src branch and identify the fields that support length, hashing, and ownership. Mark each as implementation-specific.
5. Design a text boundary that rejects invalid UTF-8 while preserving binary payloads elsewhere in the system.

## Review Questions

1. Why does a PHP string need both an explicit length and a trailing NUL in the conceptual model?
2. What does copy-on-write change about string assignment and mutation?
3. Why is `strlen()` not a Unicode character counter?
4. What is string interning useful for, and what must application code not assume about it?
5. When should a large result be streamed instead of assembled into one string?

## Summary

Zend represents PHP strings as managed, length-counted byte sequences with ownership metadata, a hash field, and a NUL-terminated buffer in current implementations. This explains binary safety, copy-on-write, hashing, and allocation concerns, but not Unicode semantics: encoding-aware behavior belongs to PHP functions, extensions, and application contracts. `zend_string` layout and interning are private, version-sensitive details; measure memory and bytes at the boundaries that matter.

## References

- [PHP Manual: strings](https://www.php.net/manual/en/language.types.string.php)
- [PHP Manual: string functions](https://www.php.net/manual/en/ref.strings.php)
- [PHP Manual: multibyte string extension](https://www.php.net/manual/en/book.mbstring.php)
- [php-src PHP-8.4: `Zend/zend_string.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_string.h)
- [php-src PHP-8.4: `Zend/zend_string.c`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_string.c)
- [php-src PHP-8.4: `Zend/zend_types.h`](https://github.com/php/php-src/blob/PHP-8.4/Zend/zend_types.h)
- [PHP Manual: streams](https://www.php.net/manual/en/book.stream.php)
