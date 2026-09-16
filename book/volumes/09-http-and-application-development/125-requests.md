---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 125
title: Requests
slug: requests
status: complete
summary: ../../_ai/chapter-summaries/125-requests-summary.md
---

# Chapter 125 — Requests

## Why This Matters

Everything a web application does begins with data supplied by a client or intermediary. The request method, target, headers, query parameters, cookies, and body all cross a trust boundary. A request parser that silently converts malformed input, accepts an unbounded body, or confuses a missing value with an empty value turns a protocol problem into an application bug.

Parse once at the edge, preserve the distinction between raw and interpreted data, then pass a small typed value into the application. Validation is not authorization: a request can be well-formed and still be forbidden for its caller.

## Request Parts and Precedence

A request has a method, target, headers, and optional content. The target may contain a path and query string; a body can carry form data, JSON, a multipart upload, or another media type. The `Content-Type` field describes how to interpret the body. The `Accept` field describes representations the client can receive. Neither field proves that the bytes actually conform to the claimed format.

Do not merge all sources into one unlabelled array. A query parameter and a JSON property with the same name have different contracts. Keep source and presence explicit:

```php
<?php

declare(strict_types=1);

/** @return array{present: bool, value: string|null} */
function queryString(array $query, string $name): array
{
    if (!array_key_exists($name, $query)) {
        return ['present' => false, 'value' => null];
    }

    $value = $query[$name];
    if (!is_string($value)) {
        throw new InvalidArgumentException("Query parameter {$name} must be scalar.");
    }

    return ['present' => true, 'value' => $value];
}
```

Using `isset()` here would treat a present `null` as absent. For query strings, PHP commonly represents repeated names as arrays, so the boundary must decide whether `tag=a&tag=b` is valid, rejected, or represented as a list. Do not let PHP's coercion rules silently turn an array into a string.

## Reading and Decoding Bodies

`php://input` exposes the request body for formats such as JSON. Read it under a configured size limit and decode with exceptions. A body is untrusted bytes until its media type and syntax have been checked:

```php
<?php

declare(strict_types=1);

/** @return array<string, mixed> */
function readJsonObject(string $body, int $maximumBytes = 1_048_576): array
{
    if ($maximumBytes < 1 || strlen($body) > $maximumBytes) {
        throw new LengthException('Request body is too large.');
    }

    try {
        $value = json_decode($body, true, flags: JSON_THROW_ON_ERROR);
    } catch (JsonException $exception) {
        throw new InvalidArgumentException('Malformed JSON.', previous: $exception);
    }

    if (!is_array($value) || array_is_list($value)) {
        throw new InvalidArgumentException('A JSON object is required.');
    }

    /** @var array<string, mixed> $value */
    return $value;
}
```

The PHP process may already be constrained by web-server and `post_max_size` settings, but application limits still provide a local contract and a useful error path. Avoid accepting a body into a string when a multipart upload or streaming protocol can exceed memory limits. For large data, use an upload mechanism that writes to controlled temporary storage and validates size, type, and content before moving it to a final location. [Chapter 133 — Uploads](133-uploads.md) covers that boundary.

JSON object keys are strings, values can be nested, and a decoded `null` is different from a missing property. Validate required keys and types after decoding. Do not use `filter_input()` as a complete request validator: filtering and business rules remain separate, and the available input source behavior depends on the server environment.

## Normalizing Headers

HTTP field names are case-insensitive, while values retain their own syntax. A framework usually gives the application a normalized header map; if you build one, normalize names to lowercase and reject malformed or ambiguous values. Do not combine security-sensitive fields by blindly joining duplicate values. `Content-Length`, `Host`, `Authorization`, and forwarding fields deserve explicit policy.

The `Host` value selects a virtual host and may influence links, tenant selection, or password-reset URLs. Accept only configured hosts. If the application sits behind a trusted proxy, allow forwarded scheme and client address fields only from that proxy and only according to the deployment's hop count or trusted network. Never make “the client sent `X-Forwarded-Proto: https`” sufficient proof on a directly reachable application server.

## A Typed Boundary

The application should receive a request object whose fields have already passed syntax checks:

```php
<?php

declare(strict_types=1);

final readonly class CreateReportRequest
{
    public function __construct(
        public string $title,
        public string $visibility,
    ) {
        if ($this->title === '' || strlen($this->title) > 200) {
            throw new InvalidArgumentException('Title has an invalid length.');
        }
        if (!in_array($this->visibility, ['private', 'team'], true)) {
            throw new InvalidArgumentException('Visibility is invalid.');
        }
    }

    /** @param array<string, mixed> $payload */
    public static function fromJson(array $payload): self
    {
        $title = $payload['title'] ?? null;
        $visibility = $payload['visibility'] ?? 'private';
        if (!is_string($title) || !is_string($visibility)) {
            throw new InvalidArgumentException('Invalid report fields.');
        }

        return new self(trim($title), $visibility);
    }
}
```

The object validates shape and local invariants. An authorization service must still decide whether the authenticated actor may create a team-visible report. Keep that decision after authentication and before the state change; do not encode it as a client-controlled field.

## Query Strings, Forms, and Repeated Values

Query values are strings at the protocol boundary, even when the application expects an integer or date. Parse with a strict rule and a range. A missing `page` can select a default; `page=` may be invalid; `page=abc` must not quietly become zero. URL decoding, plus signs, repeated keys, and Unicode normalization should be covered by tests for endpoints that rely on them.

For `application/x-www-form-urlencoded`, use the framework's parser after applying a body limit. For `multipart/form-data`, treat each part's filename, media type, and content as attacker-controlled. An uploaded filename is display metadata, not a safe filesystem path. CSRF protections and form-specific considerations appear in [Chapter 132 — Forms](132-forms.md).

## Failure and Security Modes

Return `400 Bad Request` for malformed syntax, `413 Content Too Large` when the body exceeds policy, `415 Unsupported Media Type` when the media type is not accepted, and `422 Unprocessable Content` when the syntax is valid but fields fail validation. Use a stable error format and avoid reflecting raw parser details that might contain sensitive data.

Beware parser differentials: if a reverse proxy and PHP disagree about URL decoding, duplicate fields, or body framing, an attacker may make one component authorize a different request from the one another component executes. Keep parsing rules aligned, reject ambiguity, and test the complete proxy-to-application path for high-risk routes.

## Testing Requests

Use table-driven tests for missing, empty, repeated, malformed, boundary-size, and valid values. Include content-type variations, invalid JSON encodings, nested values where only scalars are allowed, and a body exactly at the limit. Integration tests should send bytes through the HTTP server rather than directly calling `fromJson()`, because server configuration and middleware can change what PHP receives.

## Common Mistakes

- Treating every request value as a string or every string as a valid integer.
- Using a single merged array for query, body, cookies, and headers.
- Reading an unlimited body into memory.
- Treating a declared `Content-Type` as proof of the body format.
- Confusing validation with authentication or authorization.
- Trusting forwarded headers from an untrusted network peer.
- Returning detailed parser exceptions to an external client.

## Exercises

1. Implement strict parsing for a `limit` query parameter with a default, minimum, and maximum.
2. Add a request parser that accepts only `application/json` with an optional charset parameter.
3. Create tests for a missing JSON key, a JSON `null`, an empty string, and a wrong JSON type.
4. Document which proxy may set forwarded headers in a deployment and what happens when the request bypasses that proxy.

## Review Questions

1. Why should raw request data and typed application input be separate values?
2. What is the difference between malformed syntax and semantically invalid data?
3. Why is `isset()` not enough when presence and null are different states?
4. Which request values can affect URL generation and tenant selection?
5. Why must body limits exist even when the web server has a configured limit?

## Summary

Requests are untrusted protocol input. Parse method, target, fields, headers, and content separately; enforce size and media-type policies; decode with explicit errors; and validate into typed values before application logic. Authentication, authorization, and business invariants remain separate decisions.

## References

- [RFC 9110 — Request Semantics](https://www.rfc-editor.org/rfc/rfc9110.html#name-request-semantics)
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html)
- [PHP Manual: `php://input`](https://www.php.net/manual/en/wrappers.php.php)
- [PHP Manual: `json_decode`](https://www.php.net/manual/en/function.json-decode.php)
