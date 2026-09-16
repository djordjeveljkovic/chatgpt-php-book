---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 143
title: Input Validation
slug: input-validation
status: complete
summary: ../../_ai/chapter-summaries/143-input-validation-summary.md
---

# Chapter 143 — Input Validation

## Why This Matters

Every value crossing into an application has an origin, a shape, and an interpretation. Query parameters, JSON, form fields, headers, cookies, uploaded files, queue messages, and database records can be malformed, stale, surprising, or deliberately hostile. Input validation turns an untrusted value into a value the next layer can safely reason about.

Validation is not the same as escaping. Validation asks whether a value is acceptable for this operation. Encoding happens later, at the output context, so accepted text can be safely represented in HTML, SQL parameters, a URL, or a shell-free process argument. A value that passes validation is still untrusted when it is rendered.

## Parse, Normalize, Validate, and Encode

Keep the stages distinct:

1. **Parse:** decode the transport, such as JSON or a URL query string.
2. **Normalize:** apply a documented representation rule, such as trimming an email address or converting a Unicode input to the application's chosen form.
3. **Validate:** check type, size, format, relationships, authorization, and domain invariants.
4. **Map:** construct a typed command or value object with only allowed fields.
5. **Encode:** escape for the specific output context at the point of use.

Do not normalize data in a way that changes its meaning without recording the rule. Trimming a display name may be fine; trimming a password or an opaque signature is usually wrong. Avoid using a “sanitize” function as a substitute for a policy. Removing characters can turn an invalid input into a different valid value and may create collisions.

## Parse JSON Strictly

Reject an invalid content type, an oversized body, malformed JSON, and an unexpected top-level shape before invoking domain code. Use JSON_THROW_ON_ERROR so a syntax error cannot quietly become null:

~~~php
<?php

declare(strict_types=1);

/** @return array<string, mixed> */
function decodeObject(string $body, int $maxBytes = 1_048_576): array
{
    if (strlen($body) > $maxBytes) {
        throw new InvalidArgumentException('Request body is too large');
    }

    $value = json_decode($body, true, 32, JSON_THROW_ON_ERROR);
    if (!is_array($value) || array_is_list($value)) {
        throw new InvalidArgumentException('Expected a JSON object');
    }

    return $value;
}
~~~

A depth limit and body limit protect memory and CPU, but they are only part of the policy. Validate required keys, reject or explicitly ignore unknown keys, and enforce limits on arrays and strings. JSON numbers can exceed the exact range a PHP integer or downstream database type can represent; validate amounts and identifiers as strings or bounded integers when precision matters.

Avoid assigning decoded arrays directly to entities. Map an allow-list into a command:

~~~php
<?php

/** @param array<string, mixed> $input */
function createUserCommand(array $input): CreateUserCommand
{
    $name = $input['name'] ?? null;
    $email = $input['email'] ?? null;

    if (!is_string($name) || $name === '' || strlen($name) > 120) {
        throw new InvalidArgumentException('Invalid name');
    }

    if (!is_string($email) || filter_var($email, FILTER_VALIDATE_EMAIL) === false) {
        throw new InvalidArgumentException('Invalid email');
    }

    return new CreateUserCommand(trim($name), mb_strtolower(trim($email), 'UTF-8'));
}
~~~

The application still needs to define whether email comparison is case-insensitive, which Unicode normalization is supported, and whether mb_strtolower() is sufficient for the chosen identifier policy. FILTER_VALIDATE_EMAIL is a syntax check, not proof that a mailbox exists or that a user may register it.

## Scalar Types, Collections, and Relationships

Validate types before comparisons. A string “0” and integer 0 can behave differently under loose comparisons, and an absent key is different from a deliberate null. Use strict checks and bounded conversions. Validate an identifier's syntax and then authorize the resource it names; a valid UUID is not permission to read that UUID.

For arrays, define whether keys are meaningful, enforce a maximum count, and validate every element. For nested objects, validate depth and relationships such as starts_at < ends_at. Reject duplicate values where the operation requires a set. A database unique constraint remains necessary when concurrent requests can violate uniqueness after validation.

Dates and times need an explicit format, time zone, and allowed range. Parse with DateTimeImmutable::createFromFormat() and inspect errors rather than accepting a silently adjusted date. Amounts should use integer minor units or a decimal library with a defined rounding mode; binary floating point is not a money validation policy.

## Validation and Domain Invariants

Transport validation answers whether a request can be understood. Domain validation answers whether the requested operation is legal now. “The title is non-empty” may be a request rule; “a published article cannot be edited by this role” is an authorization or state invariant. Keep the latter in a domain service or database transaction so jobs and other entry points enforce it too.

Return structured, safe errors that identify fields without echoing secrets or including stack traces. Distinguish malformed syntax from a valid request with invalid domain values according to the API contract. Do not reveal whether a private identifier exists when the endpoint's visibility policy forbids enumeration.

Validation does not replace parameterized SQL, output encoding, CSRF protection, authorization, or resource limits. An input can be a valid string and still be an XSS payload in an HTML context, a valid URL that points to an internal service, or a valid path that escapes an allowed directory after decoding.

## Files, Headers, and Queues

Uploads need independent validation of size, error status, detected type, extension policy, storage name, and destination permissions; see [Chapter 150](150-file-upload-security.md). Do not trust a filename or client MIME type. Headers such as X-Forwarded-For are meaningful only when a trusted proxy chain is configured. Queue messages need a schema version, maximum size, authenticated producer policy where applicable, and a dead-letter or retry policy.

Validate before expensive work. A regex with catastrophic backtracking, deeply nested data, or an unbounded decompression step can be a denial-of-service vector even when the final value would fail validation. Prefer bounded parsers and simple allow-lists, and measure validation cost for large inputs.

## Failure and Threat Analysis

* **Type confusion:** loose comparisons accept an unexpected scalar. Decode and check exact types.
* **Mass assignment:** generic mapping lets a caller set role or ownership. Map an allow-list.
* **Normalization collision:** two inputs become one after lossy cleanup. Define canonicalization and uniqueness together.
* **Validation bypass:** a queue or CLI path skips HTTP validation. Put domain invariants below controllers.
* **Resource exhaustion:** huge strings, arrays, nesting, or regex work consume workers. Set bounds before parsing and processing.
* **Context confusion:** validation is treated as output safety. Encode for HTML, SQL parameters, URLs, or logs at the sink.
* **Race after validation:** another request changes state before the write. Use a transaction, lock, conditional update, or database constraint.

## Testing Validation

Test the acceptance boundary, not only happy paths. Include missing, null, wrong scalar type, empty and whitespace-only values, maximum and maximum-plus-one sizes, invalid UTF-8, malformed JSON, unknown fields, duplicate array values, invalid relationships, and boundary dates. Property-based or fuzz tests are useful for parsers and normalizers.

Integration tests should verify content-type and body-size handling, error status and shape, authorization after identifier validation, database constraints, and queue consumers. Assert that logs and error responses redact secrets. Add regression tests for every bypass or ambiguity discovered in production.

## Exercises

1. Define a JSON schema and PHP mapper for a reservation command. Include types, size limits, time zones, interval ordering, and unknown-field policy.
2. Write tests for a value object that accepts integer cents but rejects floats, negative amounts, overflow, and invalid currency codes.
3. Find three places where validation could be skipped in your application, such as a job, import, or admin tool. Move a domain invariant below the transport boundary.
4. Design an error response that distinguishes malformed JSON, invalid fields, and a forbidden resource without leaking existence.

## Review Questions

1. How do validation and output encoding differ?
2. Why should decoded request arrays not be hydrated directly into entities?
3. Which limits should be applied before expensive parsing or processing?
4. Why is a syntactically valid identifier not an authorization decision?
5. Where should an invariant live if HTTP and queue consumers both change the same resource?
6. Why can lossy normalization create security or data-integrity problems?

## Summary

Input validation is a typed, bounded boundary: parse, normalize by policy, validate structure and domain relationships, map an allow-list, and encode only at the output context. Use strict types, limits, safe error contracts, and database constraints; apply the same invariants to HTTP, jobs, imports, and administrative tools. Validation reduces risk but does not replace authorization, parameterized queries, output encoding, CSRF protection, or race-safe writes.

## References

- [PHP json_decode()](https://www.php.net/manual/en/function.json-decode.php)
- [PHP filter_var()](https://www.php.net/manual/en/function.filter-var.php)
- [PHP DateTimeImmutable::createFromFormat()](https://www.php.net/manual/en/datetimeimmutable.createfromformat.php)
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

