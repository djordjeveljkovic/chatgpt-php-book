---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 18
title: Strings
slug: strings
status: complete
summary: ../../_ai/chapter-summaries/018-strings-summary.md
---

# Chapter 18 — Strings

## Why This Matters

Strings cross almost every boundary in a PHP system: HTTP bodies, names, identifiers, SQL parameters, log records, file contents, tokens, passwords, and serialized messages. A string that looks like text may actually be bytes. A string that is valid UTF-8 may still be invalid for a filename, an HTML attribute, or a SQL statement. Most string bugs are boundary bugs: the program has not made encoding, normalization, length, or output context explicit.

Before manipulating a string, ask:

- Is this arbitrary bytes, ASCII protocol data, or human-readable Unicode text?
- What encoding does the boundary promise?
- Is length measured in bytes, code points, or grapheme clusters?
- Is comparison exact, case-insensitive, locale-aware, or canonicalized?
- Is this value going into HTML, SQL, a shell, a URL, a header, or a log?
- What are the maximum size and failure behavior?

There is no universal safe-string operation. Safety comes from the destination context and the contract.

## Mental Model

PHP strings are byte sequences. PHP does not attach a character encoding to a string value. ASCII-compatible text often looks correct because one ASCII character occupies one byte, but many Unicode characters occupy multiple bytes in UTF-8:

~~~php
<?php

declare(strict_types=1);

$text = 'café';

strlen($text);       // bytes, not necessarily visible characters
mb_strlen($text);    // characters, when mbstring is available and encoding is known
~~~

The source file, editor, HTTP headers, database connection, and output consumer must agree about encoding. strict_types affects type coercion; it does not make strings Unicode-aware.

Binary data is also a legitimate string. Hashes, encrypted payloads, compressed data, and file contents must not be passed through text normalization or character-case functions. Give binary and text different names and boundaries.

## Core Concept

PHP provides several literal forms:

~~~php
$single = 'No interpolation: $name';
$double = "Interpolates a variable: {$name}";
$heredoc = <<<TEXT
Long text with {$name} interpolation.
TEXT;
$nowdoc = <<<'TEXT'
Literal text with $name and no interpolation.
TEXT;
~~~

Use single quotes for short literals without interpolation, double quotes when interpolation is genuinely clearer, and heredoc/nowdoc for multi-line content. Literal syntax is not an escaping strategy for output. A string inserted into HTML still needs HTML escaping, regardless of how it was declared.

Concatenation creates a new resulting value:

~~~php
$message = 'Hello, ' . $name . '!';
$message .= ' Welcome.';
~~~

Prefer a clear formatter for structured output. Do not build SQL, shell commands, or HTML by concatenating untrusted values.

## How It Works

### Measuring and slicing

strlen() and byte-oriented functions use byte offsets. substr() can split a UTF-8 sequence if used with a character count. For human text, use mb_strlen() and mb_substr() with an explicit encoding such as UTF-8, and use the Intl extension's grapheme functions when user-perceived characters matter. Even a code-point count is not always a visible-character count: an emoji sequence or a letter plus a combining mark can contain multiple code points.

~~~php
$label = 'Zoë';

$bytes = strlen($label);
$characters = mb_strlen($label, 'UTF-8');
$prefix = mb_substr($label, 0, 2, 'UTF-8');
~~~

If the required extension or valid encoding is absent, fail at startup or return a clear error. Quietly treating arbitrary bytes as text produces corrupted output and misleading length limits.

### Search and comparison

The string search functions return positions or false:

~~~php
if (str_contains($path, '/private/')) {
    // ...
}

$position = strpos($text, 'PHP');
if ($position !== false) {
    // position zero is a valid match
}
~~~

Modern PHP includes str_contains(), str_starts_with(), and str_ends_with() for straightforward case-sensitive checks. Use stripos() or a deliberate normalization for case-insensitive matching; do not assume locale-sensitive behavior is appropriate for identifiers. For security tokens and MACs, use hash_equals() for a timing-resistant equality check when comparing a secret value with an attacker-controlled candidate.

### Formatting and parsing

Use sprintf() or vsprintf() when a format is the contract:

~~~php
$line = sprintf('Order %d: %s', $orderId, $state);
~~~

Use explode() only when the delimiter and field count are simple and controlled. For CSV, use a CSV parser. For JSON, use json_decode() with JSON_THROW_ON_ERROR and validate the resulting shape. For regular expressions, define the expected grammar, anchor when matching a complete value, and set sensible input-size limits.

String parsing is a data-structure decision. Repeatedly splitting and rejoining a large document can be O(n) per pass and create several temporary strings. A streaming parser or line iterator may better fit a large input.

## What PHP Does

PHP passes strings as values and applies copy-on-write behavior when values are assigned. The language offers byte-oriented primitives and extensions such as mbstring and Intl for higher-level text operations. It does not infer whether a string is a username, a password, UTF-8 JSON, or a file.

Functions may return false, null, an empty string, or throw an exception depending on the operation and configuration. Check the documented contract and make the failure state explicit. An empty string can be valid data; do not use truthiness when the distinction matters.

## What Zend Does

At the engine level, a PHP string stores bytes and a length. It is not a null-terminated C string in the application-level sense, so embedded null bytes can exist. Extensions may impose their own encoding or parsing rules. Temporary concatenation and transformations allocate or share string storage according to engine optimizations, but those details should be measured rather than relied on for correctness.

The runtime's byte model explains why strlen() does not count Unicode characters and why arbitrary binary data can be held in a string. Encoding-aware behavior comes from libraries and extensions, not from the basic string type.

## Minimal Example

This boundary accepts a UTF-8 display name and applies a character-count limit without confusing an empty value with a failure:

~~~php
<?php

declare(strict_types=1);

function validateDisplayName(string $input): string
{
    $name = trim($input);

    if ($name === '') {
        throw new InvalidArgumentException('Display name is required');
    }

    if (!mb_check_encoding($name, 'UTF-8')) {
        throw new InvalidArgumentException('Display name must be valid UTF-8');
    }

    if (mb_strlen($name, 'UTF-8') > 80) {
        throw new InvalidArgumentException('Display name is too long');
    }

    return $name;
}
~~~

The function still needs an output policy. Returning a valid UTF-8 value does not make it safe to put directly into HTML or a log line.

## Practical Example: Context-Specific Output

The same value requires different handling in different destinations:

~~~php
$search = $_GET['q'] ?? '';

// HTML text context:
echo htmlspecialchars($search, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');

// SQL value context: use a prepared statement parameter, not escaping.
$statement = $pdo->prepare('SELECT id FROM products WHERE name = :name');
$statement->execute(['name' => $search]);
~~~

HTML escaping is not SQL escaping. URL encoding is not HTML escaping. Shell escaping is not a substitute for avoiding a shell. A security review follows the value to its sink and chooses the sink's mechanism.

For a search feature, decide whether matching is exact, prefix, token, or full text. A lowercased PHP scan may be acceptable for a small in-memory list, but a database index or search engine is usually the right owner for large durable data. Case folding and accent handling can be domain-specific; document the policy rather than using strtolower() as a universal solution.

## Production Example: Framing a Line Protocol

A worker that reads newline-delimited messages must distinguish framing from content:

~~~php
function readLine(Stream $stream, int $maximumBytes): string
{
    $line = $stream->readUntil("\n", $maximumBytes + 1);

    if ($line === null || strlen($line) > $maximumBytes) {
        throw new RuntimeException('Message is missing or too large');
    }

    return rtrim($line, "\r\n");
}
~~~

The important decisions are the maximum size, line-ending policy, and behavior when the peer closes the connection mid-message. Trimming all whitespace would silently change a meaningful payload; removing only framing characters preserves content. A production protocol also needs timeouts, authentication, and a strategy for malformed messages.

## Bad Example

~~~php
<?php

$name = $_POST['name'] ?? '';
$preview = substr($name, 0, 20);
echo '<h1>' . $preview . '</h1>';
~~~

This code can cut a UTF-8 sequence, allows HTML injection, accepts an unbounded request before slicing, and assumes a form field is text with the expected encoding. It also places input acquisition and output rendering in one operation, making the failure policy hard to test.

## Better Example

~~~php
<?php

declare(strict_types=1);

function renderNameHeading(mixed $rawName): string
{
    if (!is_string($rawName)) {
        throw new InvalidArgumentException('name must be a string');
    }

    $name = validateDisplayName($rawName);
    $shortName = mb_substr($name, 0, 20, 'UTF-8');

    return '<h1>' . htmlspecialchars(
        $shortName,
        ENT_QUOTES | ENT_SUBSTITUTE,
        'UTF-8',
    ) . '</h1>';
}
~~~

The boundary validates type and encoding, applies a character policy, and escapes for the actual output context. A framework template engine may provide the final escaping adapter, but the rule remains the same.

## Edge Cases

- strlen() counts bytes; it is correct for protocol limits measured in bytes and wrong for many user-visible character limits.
- substr() uses byte offsets. Do not use it to truncate Unicode text unless byte truncation is explicitly intended.
- Empty string, string "0", null, and false are distinct values. Avoid truthiness when parsing form data or identifiers.
- A trailing newline can be framing or meaningful content. Remove only what the protocol defines.
- trim() removes a set of whitespace characters; it is not a general Unicode normalization operation.
- Unicode can have multiple canonically equivalent representations. Normalize when the business rule requires canonical comparison, using an appropriate library.
- Regular expressions require a delimiter, pattern, and encoding policy. Invalid UTF-8 with a UTF-8 pattern can fail rather than match.
- A string may contain null bytes. Validate before passing it to filesystem, process, or C-library boundaries.
- Header values have strict grammar and response-splitting risks. Never copy arbitrary input into a header without validation.

## Performance

Most basic scans, searches, and transformations are O(n) in the byte length. Repeated concatenation in a loop can create avoidable temporary data; collect bounded pieces and join once when profiling shows a problem. A regular expression may have much higher behavior for pathological patterns and inputs, so constrain both.

Character-aware operations inspect encoding units and can cost more than byte operations. That cost is usually correct for human text. For large files, avoid loading the entire string when a stream, iterator, or chunked hash is sufficient. Measure peak memory, not just elapsed time, because a request can fail from memory exhaustion before it reaches a CPU limit.

## Security

Validate size before expensive decoding or regular-expression processing. Use allow-lists for identifiers and protocol fields. Use prepared statements for SQL values, dedicated escaping for HTML, rawurlencode() or a URL builder for URL components, and a process API that avoids a shell for commands. Never log passwords, session tokens, API keys, or full authorization headers.

Canonicalization is a security boundary. Validate the representation that will actually be used: decode once, normalize only under an explicit policy, then authorize and persist that canonical value. Be cautious with Unicode confusables in usernames, filenames, and administrator-facing identifiers. Reject or isolate malformed input rather than repairing it invisibly.

## Database Interaction

Use the database connection's configured character set and collation deliberately. A column collation controls comparison and ordering; it is not interchangeable with application-side lowercasing. Bind string values as parameters and let the driver handle protocol encoding. When persisting JSON or serialized text, validate size and shape before writing and version the format if it crosses deployments.

Search queries need a data-model decision. A leading-wildcard pattern, application-side scan, or unindexed case conversion may prevent an index from helping. Check the query plan and measure cardinality before adding a PHP loop that transfers all candidate rows.

## Concurrency

Strings are process-local values. A shared counter, lock token, or version string is not made atomic by being stored in a PHP variable. When a string represents an optimistic-lock version, idempotency key, or message identifier, enforce uniqueness and compare-and-set semantics at the shared store. For secret comparisons, use constant-time comparison where appropriate, but do not mistake it for authorization or replay prevention.

## Testing

Test bytes and text separately:

~~~php
public function testDisplayNameUsesCharacterLimit(): void
{
    self::assertSame('Zoë', validateDisplayName(' Zoë '));
}

public function testHeadingEscapesHtml(): void
{
    self::assertSame(
        '<h1>&lt;script&gt;</h1>',
        renderNameHeading('<script>'),
    );
}
~~~

Include ASCII, multi-byte text, combining marks, emoji sequences, malformed UTF-8, empty strings, the string "0", embedded null bytes, boundary lengths, line endings, and hostile HTML. Test each output sink with its own assertion; one test for “a string exists” does not prove it is safe for SQL or HTML. Add fuzz or property tests for parsers and size-limit tests for regular expressions.

## Common Mistakes

- Treating a byte string as a Unicode character sequence.
- Using one escaping function for every output context.
- Checking if ($value) when empty string or "0" is valid.
- Comparing secrets with ordinary equality in a security-sensitive path.
- Using string concatenation to construct SQL or shell commands.
- Applying strtolower() as a universal internationalized comparison policy.
- Truncating user text by bytes and producing invalid UTF-8.
- Reading an unbounded request body into memory before checking its size.

## Senior Engineer Thinking

A string variable should have a contract: encoding, normalization, maximum size, permitted grammar, and destination contexts. Put conversion at boundaries and keep domain code working with validated values. When a string is really an identifier, money amount, date, or structured message, a dedicated type or parser can prevent repeated ad hoc handling.

The senior question is not “which string function is shortest?” It is “what are these bytes, who owns their interpretation, and what can go wrong when they cross the next boundary?” That question prevents both correctness bugs and injection vulnerabilities.

## Exercises

1. Implement a UTF-8 display-name validator with byte and character limits. Explain why both limits may be necessary.
2. Write a parser for a three-field, newline-delimited protocol. Define framing, maximum size, malformed-input behavior, and tests.
3. Create a table showing how the same input must be handled for HTML text, an HTML attribute, a URL query value, SQL, and a shell-free process invocation.
4. Benchmark whole-file loading versus chunked hashing for a large file and record peak memory.

## Review Questions

1. Why does PHP's basic string type not imply Unicode text?
2. When is strlen() the correct operation, and when is it misleading?
3. Why is HTML escaping not a replacement for SQL parameters?
4. What does hash_equals() protect, and what does it not protect?
5. Why can canonicalization change the security meaning of validation?
6. What failure modes appear when a line protocol has no maximum message size?
7. Why should a large text file often be processed as a stream?

## Summary

PHP strings are byte sequences; text encoding and output context are separate contracts. Use byte functions for byte protocols, encoding-aware libraries for Unicode, and explicit parsers for structured data. Validate size and grammar before expensive work, escape or parameterize at the destination sink, and test malformed, multi-byte, boundary, and security-sensitive inputs. When strings represent richer concepts, give those concepts a stronger type or parser.
