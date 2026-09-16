---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 124
title: HTTP
slug: http
status: complete
summary: ../../_ai/chapter-summaries/124-http-summary.md
---

# Chapter 124 — HTTP

## Why This Matters

An HTTP endpoint is a protocol participant. It receives a request whose method, target, headers, and content have defined meanings, then emits a response whose status, fields, and content form a contract with a client, proxy, cache, or browser. A framework can make this feel like a controller method, but production failures often happen at the protocol boundary: a retry repeats a non-idempotent action, a proxy changes the apparent scheme, or a response is cached under the wrong key.

HTTP is deliberately stateless at the protocol level. A server may use cookies, sessions, or tokens to associate requests with an application identity, but each message must still be interpretable on its own. RFC 9110 defines the shared semantics for HTTP versions; HTTP/1.1, HTTP/2, and HTTP/3 differ in framing and transport, while the method and status concepts remain largely common.

## The Message Model

Conceptually, a request contains control data, a header section, and optional content:

```text
GET /reports/42?format=json HTTP/1.1
Host: example.test
Accept: application/json

[no content]
```

The response has analogous parts:

```text
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 34

{"id":42,"title":"Quarterly report"}
```

The blank line separates fields from content. HTTP/2 and HTTP/3 do not send these literal lines on the wire, but the application still reasons about the same fields and semantics. Framing is the transport concern; content is the representation transferred after framing has been interpreted. A `Content-Length` value therefore describes octets, not PHP characters or decoded JSON values.

The request target identifies the resource and operation context. The `Host` field (or the authority component in newer versions) selects the destination at a shared IP address. A URI is not automatically a filesystem path: route it through an explicit mapping and decode path components according to the routing contract.

## Methods and Their Contracts

The method communicates intent. `GET` retrieves a representation, `POST` asks the server to process content or create a subordinate resource, `PUT` replaces a representation at a known target, `PATCH` applies a partial change, and `DELETE` requests removal. `HEAD` has the semantics of `GET` without response content and is useful for metadata checks.

The terms **safe** and **idempotent** describe protocol semantics, not whether an implementation has zero side effects. `GET` is safe: an automated client may fetch it without intending a state change. A logged access or a cache miss can still have server-side effects. `PUT` and `DELETE` are idempotent: repeating the same request has the same intended effect as making it once, although logging or timestamps may differ. `POST` is not generally idempotent. An application can add an idempotency key and a storage design that makes a particular POST operation safely retryable; the method alone does not provide that guarantee. [Chapter 141 — Idempotency](141-idempotency.md) develops that design.

Do not use `GET` for an action that changes account state merely because an HTML link is convenient. Crawlers, prefetchers, caches, and browser history can cause a safe request to be repeated. Treat method selection as part of authorization and abuse prevention.

## Status Classes and Representation

A final response has a three-digit status code. The first digit identifies the class: `1xx` is informational, `2xx` successful, `3xx` redirection, `4xx` client error, and `5xx` server error. Clients must understand the class even when they do not know a particular code. Use the most specific established code that describes the contract: `201 Created` with a `Location` for a newly created resource, `204 No Content` when there is no representation to return, `404 Not Found` when the target is not available, and `409 Conflict` when a current resource state prevents the requested operation.

A status code does not replace a useful representation. An API error can contain a stable machine-readable type, a human-readable detail safe for the caller, and a request or trace identifier. Do not serialize exception messages, SQL fragments, stack traces, or secrets into production responses. The same failure may need HTML for a browser and JSON for an API; negotiate the representation rather than returning a JSON blob with an HTML content type.

## The PHP Boundary

PHP receives server-provided values in predefined variables, but their presence and trust level depend on the web server and deployment. `$_SERVER['REQUEST_METHOD']`, `$_SERVER['REQUEST_URI']`, and selected `HTTP_*` fields are input, not authenticated facts. Normalize and validate them at one boundary.

The following small dispatcher demonstrates the protocol decision without coupling it to a framework:

```php
<?php

declare(strict_types=1);

final class HttpRequest
{
    /** @param array<string, string> $headers */
    public function __construct(
        public readonly string $method,
        public readonly string $target,
        public readonly array $headers,
        public readonly string $body,
    ) {
    }
}

function methodNotAllowed(HttpRequest $request): never
{
    header('Allow: GET, HEAD');
    http_response_code(405);
    echo 'Method not allowed';
    exit;
}

function dispatch(HttpRequest $request): void
{
    $path = parse_url($request->target, PHP_URL_PATH);
    if (!is_string($path)) {
        http_response_code(400);
        echo 'Invalid request target';
        return;
    }

    if ($path !== '/reports/42') {
        http_response_code(404);
        echo 'Not found';
        return;
    }

    if ($request->method !== 'GET' && $request->method !== 'HEAD') {
        methodNotAllowed($request);
    }

    header('Content-Type: application/json; charset=utf-8');
    $payload = json_encode(['id' => 42, 'title' => 'Quarterly report'], JSON_THROW_ON_ERROR);
    http_response_code(200);
    if ($request->method === 'GET') {
        echo $payload;
    }
}
```

In a real application, the web server or framework owns parsing, body limits, routing, and response emission. The example makes two boundaries explicit: the path is parsed as a URI component, and `HEAD` shares the metadata of `GET` without writing the body. A `HEAD` response still needs correct headers such as content type and, when known, representation length.

## Failure, Security, and Operations

HTTP failures cross process and network boundaries. A client can disconnect after the server commits a database transaction, a proxy can retry a request, or an upstream timeout can leave the application unsure whether the upstream completed. Set timeouts at every outbound hop, classify failures, and make retries conditional on method semantics and operation design. Log a request identifier and outcome without logging authorization headers, cookies, or raw sensitive bodies.

Request smuggling and cache poisoning exploit disagreement between intermediaries about parsing or cache keys. Keep the web server and application server patched and configured consistently, reject ambiguous framing, validate the effective host and scheme through a trusted proxy configuration, and include the right variation fields in cache policy. HTTPS protects transport in transit; it does not make an unvalidated `Host` field or an unsafe redirect trustworthy.

## Testing the Contract

Test at the HTTP boundary, not only by calling a controller function. Verify method handling, a malformed target, a missing resource, a successful representation, a `HEAD` response without content, and an unsupported method with an `Allow` field. Test the behavior through the real server or an HTTP test client so status, headers, encoding, and body framing are observed together. Add a proxy or integration test when trusted-forwarded headers affect URL generation or security decisions.

## Common Mistakes

- Treating HTTP/2 or HTTP/3 as a different application API instead of a different wire framing.
- Choosing `GET` for a state-changing action.
- Assuming a `2xx` status means every requested sub-operation succeeded.
- Trusting `$_SERVER` values or forwarded headers without a deployment trust policy.
- Returning exception details or credentials in an error representation.
- Retrying a request without considering duplication and commit uncertainty.
- Setting a content type that does not describe the actual bytes sent.

## Exercises

1. Design the request and response contract for creating a report, including method, success status, validation failure, and duplicate submission behavior.
2. Add conditional handling for `OPTIONS` to the dispatcher and explain which parts belong to the application and which to a CORS policy.
3. Write an integration test proving that `HEAD /reports/42` returns the same representation headers as `GET` but an empty body.
4. Draw the request path through a browser, CDN, reverse proxy, PHP-FPM, and database, then mark where a timeout or retry can occur.

## Review Questions

1. Which parts of an HTTP message are semantic content and which parts are transport framing?
2. Why is idempotence useful for retries but insufficient to guarantee no side effects?
3. Why must a `GET` endpoint remain safe even when a server records access logs?
4. What makes forwarded scheme and host values different from directly observed connection properties?
5. Which tests would detect a mismatch between a response body and its `Content-Type`?

## Summary

HTTP is a contract over requests and responses. Methods express intent, status classes communicate outcome, headers describe metadata and control processing, and content carries a representation. PHP applications must validate the server-provided boundary, choose safe retry behavior, emit accurate responses, and test the complete protocol contract.

## References

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [PHP Manual: `$_SERVER`](https://www.php.net/manual/en/reserved.variables.server.php)
- [PHP Manual: `header`](https://www.php.net/manual/en/function.header.php)
