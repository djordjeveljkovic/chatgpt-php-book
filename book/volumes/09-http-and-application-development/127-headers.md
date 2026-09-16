---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 127
title: Headers
slug: headers
status: complete
summary: ../../_ai/chapter-summaries/127-headers-summary.md
---

# Chapter 127 — Headers

## Why This Matters

HTTP headers are small fields with large effects. They select a representation, control caching, carry authentication challenges, describe framing, influence browser security, and communicate proxy behavior. A missing `Vary` can serve one user's language to another; a reflected header can enable response splitting; an incorrect `Content-Type` can make a browser interpret attacker-controlled bytes as executable content.

Treat headers as typed protocol data with an allowlist of names and value rules. Header names are case-insensitive, but their semantics and permitted values are field-specific.

## Header Fields and Combination

A message has a header section before content. Some fields can appear more than once and have defined combination rules; others, such as `Content-Length` and `Host`, require a single unambiguous value. Never join duplicate security-sensitive fields with a comma merely because a generic map makes that convenient. Reject conflicting framing or authority fields at the earliest trusted parser.

Header values are not automatically safe strings. A value containing carriage return or line feed can split a response into new fields when passed to a low-level emitter. Reject controls before calling `header()`, and use framework APIs that validate names and values. User input should rarely be copied into a response field at all.

## Representation and Negotiation

`Content-Type` describes the selected representation. `Accept` tells a server what response media types a client prefers; it is not a command to return arbitrary content. `Accept-Language` and `Accept-Encoding` participate in content selection and compression. When a response changes based on a request field, `Vary` tells a shared cache which field contributes to the cache key.

Negotiation needs a deterministic fallback. If the client explicitly refuses every supported type, return `406 Not Acceptable` or apply the documented API default. Do not infer that a missing `Accept` means “JSON only.” For APIs, a versioned media type can be clearer than a custom header, but whichever contract is selected must be documented and tested.

## Caching Fields

`Cache-Control` is the primary cache policy field. `public`, `private`, `no-store`, `max-age`, `s-maxage`, and `must-revalidate` have different implications for browser and shared caches. `Expires` is an older absolute-date mechanism and should not be used to override an explicit modern policy. `ETag` identifies a representation version for conditional requests; a strong tag can support byte-level equivalence, while a weak tag indicates semantic equivalence only.

The cache key is not automatically “URL plus method.” It may include selected request fields, authorization context, and content encoding according to the cache and `Vary` policy. Never mark a personalized response `public` without an explicit privacy design. [Chapter 126 — Responses](126-responses.md) applies these fields when constructing the result.

## Security and Cross-Origin Fields

Browser-facing responses often need `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, and a framing policy such as `Content-Security-Policy: frame-ancestors`. These headers reduce classes of browser attacks, but they do not replace output encoding or authorization.

CORS is a browser policy negotiated with fields such as `Origin`, `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, and `Access-Control-Allow-Headers`. If the allowed origin is selected dynamically, include `Vary: Origin` when a shared cache can store the response. Never pair a wildcard origin with credentials. Validate allowed origins against a configured list; suffix matching such as `endsWith('example.com')` can allow `attackerexample.com`.

Authentication headers require strict handling. `Authorization` is a request credential and should be redacted from logs. `WWW-Authenticate` describes a `401` challenge; it is not the same as `403`. Cookies have their own attributes and security rules and are covered in [Chapter 128 — Cookies](128-cookies.md).

## Forwarded Headers and Proxies

`Forwarded`, `X-Forwarded-For`, `X-Forwarded-Proto`, and similar fields describe proxy observations, not cryptographic facts. A reverse proxy may add them, but a directly connected client can also send arbitrary values unless the edge strips and rewrites them. Configure a trusted proxy boundary, define how many hops are trusted, and use the resulting normalized value only for the decisions it supports. Do not use an untrusted forwarded address for access control or rate-limit identity.

## A Safe Header Map

Keep response header creation centralized. An allowlist prevents arbitrary application code from emitting transport or security fields:

```php
<?php

declare(strict_types=1);

/** @param array<string, string> $headers */
function validateResponseHeaders(array $headers): array
{
    $allowed = [
        'cache-control',
        'content-location',
        'content-type',
        'etag',
        'location',
        'retry-after',
        'vary',
        'x-content-type-options',
    ];
    $validated = [];

    foreach ($headers as $name => $value) {
        $normalized = strtolower($name);
        if (!in_array($normalized, $allowed, true)) {
            throw new InvalidArgumentException("Header is not allowed: {$name}");
        }
        if ($name === '' || preg_match('/[\x00-\x1F\x7F]/', $name . $value) === 1) {
            throw new InvalidArgumentException('Header contains a control character.');
        }
        if ($normalized === 'content-type' && trim($value) === '') {
            throw new InvalidArgumentException('Content-Type cannot be empty.');
        }
        $validated[$name] = $value;
    }

    return $validated;
}
```

An allowlist must reflect the application's needs and should be paired with field-specific validation. It is not a complete HTTP parser. For example, `Location` should be validated as an allowed redirect target, and `Retry-After` should be either a valid delay or an HTTP date according to the API contract. Header names are normalized for comparison, while the emitted spelling can be selected by the response layer.

## Framing, Compression, and Trailers

`Content-Length` counts the final content bytes under the applicable message framing. If a proxy compresses the response, it must update framing for the compressed bytes. An application should not set a length it cannot know after downstream transformations. HTTP/1.1 chunked framing and HTTP/2 or HTTP/3 stream framing are transport concerns; do not put chunk syntax into the application body.

Trailer fields arrive after streamed content and are useful only when the client and intermediaries support the declared trailer contract. They cannot repair a response whose status or primary headers were wrong. For most PHP APIs, ordinary headers before content are easier to observe and cache.

## Failure and Observability

Record selected status, route, response size, cache outcome, and a request identifier. Redact credentials and personal data, and avoid logging all header values by default. A useful log can say that `Authorization` was present without copying its token. Track rejected headers and malformed forwarding data as security-relevant metrics without turning attacker-controlled values into log lines.

Header changes should be reviewed with the full path: browser, CDN, load balancer, web server, PHP runtime, and application. A field emitted by PHP may be stripped, duplicated, or rewritten downstream. Capture integration traffic in a controlled environment when a cache or proxy is part of the contract.

## Testing Headers

Test field names case-insensitively, duplicate handling, control-character rejection, `Vary` behavior, cache privacy, CORS allowlists, and trusted versus untrusted proxy paths. Assert that credentials do not appear in logs. Use an HTTP client against the actual deployment topology for compression and cache tests; a unit test of a PHP array cannot reveal a proxy that rewrites `Cache-Control`.

## Common Mistakes

- Treating headers as arbitrary string metadata.
- Reflecting user input into a header without control-character validation.
- Caching a response that varies by authorization or origin.
- Trusting `X-Forwarded-For` from every client.
- Pairing CORS credentials with a wildcard origin.
- Setting `Content-Length` before compression or transfer transformations.
- Logging authorization and cookie values.

## Exercises

1. Design a cache policy for a public article and a user-specific dashboard, including `Vary` where needed.
2. Add an origin allowlist that rejects look-alike domains and test preflight and credentialed requests.
3. Build a trusted-proxy parser for a documented one-hop deployment and test direct-client spoofing.
4. Capture a compressed response and explain which component owns the final `Content-Length`.

## Review Questions

1. Why are header names case-insensitive but header values field-specific?
2. What does `Vary` protect in a shared cache?
3. Why cannot an application trust forwarded headers merely because they use a conventional name?
4. What is the difference between `WWW-Authenticate` and an authorization decision?
5. Why should application code avoid setting `Content-Length` when a proxy can transform content?

## Summary

Headers control interpretation, caching, authentication, security, and proxy behavior. Validate them with field-specific rules, reject ambiguity and control characters, define trusted proxy boundaries, and test the complete path through caches and intermediaries. Treat privacy and observability as part of the header contract.

## References

- [RFC 9110 — Field Definitions](https://www.rfc-editor.org/rfc/rfc9110.html#name-field-definitions)
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [Fetch Standard — CORS protocol](https://fetch.spec.whatwg.org/#http-cors-protocol)
- [OWASP: HTTP Headers Cheat Sheet](https://owasp.org/www-project-secure-headers/)
- [PHP Manual: `header`](https://www.php.net/manual/en/function.header.php)
