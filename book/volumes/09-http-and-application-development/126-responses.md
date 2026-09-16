---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 126
title: Responses
slug: responses
status: complete
summary: ../../_ai/chapter-summaries/126-responses-summary.md
---

# Chapter 126 — Responses

## Why This Matters

A response is an externally visible result. Its status code, headers, and bytes tell a client whether an operation succeeded, whether it may retry, how to interpret the content, and whether a cache can reuse it. Calling `echo` from a controller without a response contract makes these decisions accidental and makes failures difficult to test.

Construct a response as data, then emit it once at the edge. This keeps domain code independent of PHP's output buffer and prevents a late exception from producing a half-JSON, half-error response.

## Status Codes Describe Outcomes

Choose a status based on the operation's contract. `200 OK` returns a representation, `201 Created` confirms creation and commonly includes `Location`, `202 Accepted` means processing has been accepted but is not complete, and `204 No Content` confirms success without a representation. `400 Bad Request` describes malformed syntax, `401 Unauthorized` means credentials are required or invalid, `403 Forbidden` means the server will not authorize the action, and `404 Not Found` hides or reports an unavailable target according to the resource policy.

`409 Conflict` is useful when the current resource state prevents an otherwise valid operation. `412 Precondition Failed` reports a failed conditional request, while `429 Too Many Requests` communicates rate limiting and can include `Retry-After`. `500` indicates an unexpected server failure; do not return it for a user-correctable validation error. The response body may explain the failure, but the status remains the primary machine-readable signal.

## Representation and Encoding

`Content-Type` describes the media type and parameters of the bytes sent. JSON responses should use `application/json; charset=utf-8` when that is the representation contract. Encode with `JSON_THROW_ON_ERROR`; silently returning `false` or a partial value turns an internal encoding problem into invalid protocol output.

HTML output needs context-aware escaping. Escaping for element text is not the same as escaping for an attribute, a URL, JavaScript, or CSS. Set a restrictive content security policy where appropriate, but use correct output encoding as the primary defense. Never interpolate untrusted data into a `Location` header without validating the redirect destination.

For an API error, use a stable envelope:

```php
<?php

declare(strict_types=1);

final readonly class Response
{
    /** @param array<string, string> $headers */
    public function __construct(
        public int $status,
        public array $headers,
        public string $body,
    ) {
        if ($status < 100 || $status > 599) {
            throw new InvalidArgumentException('Invalid HTTP status.');
        }
    }
}

function jsonResponse(int $status, array $payload): Response
{
    return new Response(
        status: $status,
        headers: [
            'Content-Type' => 'application/json; charset=utf-8',
            'X-Content-Type-Options' => 'nosniff',
        ],
        body: json_encode($payload, JSON_THROW_ON_ERROR),
    );
}

function validationError(string $requestId, array $fields): Response
{
    return jsonResponse(422, [
        'type' => 'https://example.test/problems/validation',
        'title' => 'Validation failed',
        'status' => 422,
        'detail' => 'One or more fields are invalid.',
        'request_id' => $requestId,
        'fields' => $fields,
    ]);
}
```

The `Response` value makes the body and status testable without sending output. A production response type should define duplicate-header behavior, bodyless statuses, and whether header names are normalized. The example's payload shape is an application contract; clients should not have to parse human prose to identify an error.

## Emitting Once

PHP's `header()` and `http_response_code()` affect the outgoing response only before headers have been sent. Any preceding output, including an accidental byte-order mark or warning, may commit headers. Use a framework response emitter or an output-buffer policy, and treat warnings in response construction as failures. A simple emitter is:

```php
<?php

declare(strict_types=1);

function emit(Response $response): void
{
    if (headers_sent($file, $line)) {
        throw new LogicException("Response already started at {$file}:{$line}");
    }

    http_response_code($response->status);
    foreach ($response->headers as $name => $value) {
        header($name . ': ' . $value, replace: true);
    }

    if (!in_array($response->status, [204, 304], true)) {
        header('Content-Length: ' . strlen($response->body), replace: true);
        echo $response->body;
    }
}
```

The bodyless status list is a simplified application boundary. HTTP also defines responses such as `HEAD` responses that omit content while preserving representation metadata. A production emitter should account for the request method and server-specific transfer framing. Do not set `Content-Length` to `mb_strlen()`; framing counts encoded bytes, and `strlen()` measures bytes in the already encoded string.

## Redirects, Caching, and Conditional Responses

Redirect status codes have different method and caching implications. A temporary redirect should not accidentally turn a state-changing operation into a `GET` through client-specific rewriting. Use `Location` with an absolute or well-defined relative target, validate external redirects, and choose permanent status only when the change is durable.

Caching is a response contract. `Cache-Control: private` is appropriate for user-specific content; `no-store` prevents storage where sensitive data must not persist; `ETag` and `Last-Modified` support conditional requests. A `304 Not Modified` response has no content and is valid only when the client's validator matches the current representation under the resource's cache policy. If the response varies by authorization, locale, encoding, or another request field, declare the variation or avoid shared caching.

## Streaming and Large Responses

Encoding a large result with `json_encode()` builds the complete representation in memory. For exports or event streams, stream in bounded chunks and define failure behavior when the client disconnects. Once bytes have been sent, changing the status to an error is impossible; write durable work to a job and return `202` when the client should poll or follow a status resource. [Chapter 137 — SSE](137-sse.md) covers a long-lived streaming response.

Compression and transfer framing may be applied by the web server or proxy. Do not manually calculate a length before compression unless your application owns the final bytes. Avoid flushing every tiny chunk: it increases syscall and network overhead, and buffering behavior belongs to the complete deployment path.

## Failure and Security

Separate the internal exception from the public problem representation. Log the exception with a request identifier, return a generic `500`, and ensure the status is not accidentally overwritten by a later helper. Do not include stack traces, SQL, filesystem paths, tokens, or upstream response bodies in a public error.

Set security headers deliberately: `Content-Security-Policy` for browser content, `X-Content-Type-Options: nosniff` for type confusion protection, and suitable framing and referrer policies for the application. These fields are not substitutes for authorization, CSRF protection, or output encoding.

## Testing Responses

Unit-test response construction for status, exact media type, error shape, and body bytes. Integration-test emission for headers-before-body, `HEAD`, `204`, redirects, conditional requests, compression ownership, and large or disconnected responses. Assert that sensitive exception details do not appear in public bodies. A snapshot test of JSON alone will not catch a wrong status or cache header.

## Common Mistakes

- Returning `200` for every outcome and forcing clients to parse prose.
- Calling `header()` after output has started.
- Computing `Content-Length` from characters rather than encoded bytes.
- Sending a body with `204` or `304`.
- Caching personalized content in a shared cache.
- Building a multi-megabyte response in memory without measuring the limit.
- Leaking exception details in JSON error responses.

## Exercises

1. Implement a response for creating a report with `201`, `Location`, and a JSON representation.
2. Add an emitter test that proves a UTF-8 body's `Content-Length` equals its byte length.
3. Design conditional GET handling with an ETag and list the inputs that must affect the tag.
4. Change a long export endpoint to return `202` and a status URL, then define its failure response.

## Review Questions

1. Why is the status code part of the API contract rather than presentation?
2. When should `202 Accepted` be used instead of `200 OK`?
3. Why can an application not change a status after streaming response bytes?
4. What does a cache need to know when a representation varies by request headers?
5. Why is `strlen()` correct for a pre-encoded response body but `mb_strlen()` is not?

## Summary

Responses should be constructed as typed data and emitted once. Select status codes that describe the outcome, encode a representation with an accurate media type, define caching and redirect behavior, and protect error responses from information leaks. Test status, headers, bytes, and streaming boundaries together.

## References

- [RFC 9110 — Status Codes](https://www.rfc-editor.org/rfc/rfc9110.html#name-status-codes)
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [PHP Manual: `header`](https://www.php.net/manual/en/function.header.php)
- [PHP Manual: `http_response_code`](https://www.php.net/manual/en/function.http-response-code.php)
