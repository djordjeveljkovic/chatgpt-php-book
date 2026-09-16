---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 230
title: HTTP Performance
slug: http-performance
status: complete
summary: ../../_ai/chapter-summaries/230-http-performance-summary.md
---

# Chapter 230 — HTTP Performance

## Why This Matters

HTTP performance is the time and capacity cost of the complete request path: client, DNS, connection setup, TLS, proxy, PHP worker, database, downstream calls, response serialization, and transfer. Optimizing only PHP execution can leave users waiting on a slow query, a large payload, or a connection that cannot reach the origin.

Define a latency budget and measure each phase. Tail latency and error rate matter alongside average response time. A smaller response that loses cache correctness or a more aggressive timeout that creates duplicate commands is not a safe optimization.

## Budget the Request

Set budgets for DNS and connection setup, origin processing, dependencies, and response transfer. A request with a 500 ms user budget cannot spend 400 ms in one provider and still reliably render on a slow network. Pass a deadline or remaining budget to downstream clients rather than giving every call an independent full timeout.

PHP-FPM workers are occupied while application code runs and often while it waits on I/O. A slow downstream call reduces worker capacity even when CPU is idle. Queue non-interactive work and bound concurrency for expensive endpoints.

## Requests and Connections

Connection reuse avoids repeated TCP and TLS handshakes. Keep-alive, HTTP/2 multiplexing, HTTP/3, DNS caching, and TLS session behavior depend on client, proxy, server, and network configuration. Measure the deployed path; a protocol feature enabled at one hop may not apply end to end.

Use connection and total timeouts, verify certificates, and limit redirects and response sizes for outbound calls. Reusing one client can improve connection pooling, but shared mutable headers, credentials, and request state are unsafe across tenants or long-lived jobs.

## Response Size and Serialization

Send the representation a client needs. Select fields, paginate collections, avoid embedding unbounded relationships, and compress suitable text responses. Compression trades CPU and memory for fewer bytes; measure based on payload size and client distribution.

~~~php
<?php

declare(strict_types=1);

function jsonResponse(array $data): string
{
    return json_encode(
        $data,
        JSON_THROW_ON_ERROR
        | JSON_UNESCAPED_SLASHES
        | JSON_UNESCAPED_UNICODE,
    );
}

/** @param iterable<array<string, mixed>> $rows */
function streamJson(iterable $rows): Generator
{
    yield '[';
    $first = true;

    foreach ($rows as $row) {
        if (!$first) {
            yield ',';
        }
        $first = false;
        yield jsonResponse($row);
    }

    yield ']';
}
~~~

Streaming avoids building one large response string, but the web server and proxy may buffer it and the client must support incremental data. A JSON array is not parseable until its closing bracket arrives; newline-delimited JSON or pagination may be more useful for incremental consumers. Bound row count and output duration.

## Caching and Validators

Use cache headers to make freshness and privacy explicit. Public immutable assets can have long cache lifetimes and content hashes. A public API representation needs a cache key that includes every representation-varying input. Authenticated responses require private directives or a cache that isolates the authenticated context.

ETag and Last-Modified validators let a client revalidate without transferring an unchanged body:

~~~php
<?php

declare(strict_types=1);

function conditionalResponse(
    string $body,
    string $etag,
    ?string $ifNoneMatch,
): array {
    if ($ifNoneMatch === $etag) {
        return [
            'status' => 304,
            'headers' => ['ETag' => $etag],
            'body' => '',
        ];
    }

    return [
        'status' => 200,
        'headers' => [
            'ETag' => $etag,
            'Cache-Control' => 'private, max-age=60',
        ],
        'body' => $body,
    ];
}
~~~

A production implementation must parse validator lists and weak validators according to HTTP semantics and must ensure that the ETag represents the selected representation. A 304 response has no body; clients use their cached representation. Do not use a validator as authorization or freshness proof for mutable business state.

## Browser and Asset Performance

Reduce blocking work and payload size without weakening security. Serve appropriately sized images, cache immutable assets, avoid shipping unused JavaScript, and use compression and modern formats where the client support policy allows. Critical CSS and script ordering affect rendering, while CSP, integrity, and cookie policy remain security requirements.

Measure user-centric signals such as time to first byte, first contentful paint, largest contentful paint, interaction delay, and transfer size where the product needs browser performance. A fast origin can still produce a slow user experience on a high-latency mobile network.

## Downstream Calls and Batching

An API endpoint that calls five providers serially accumulates latency and can fail partially. Parallel calls can reduce wall time but increase connection and concurrency pressure. Use only independent calls in parallel, propagate a deadline, and define partial response behavior.

Batch requests when a provider supports them, cache stable data, and move enrichment to an asynchronous read model when immediate consistency is not required. Avoid a browser making one HTTP request per table row; provide a bounded API projection.

## Streaming, Uploads, and Large Transfers

Stream uploads to controlled temporary storage, enforce content-length and total-size limits, validate type and authorization, and clean up abandoned files. For large downloads, use a file-serving or object-storage path that supports range requests and offloads bytes from PHP workers. An application endpoint should authorize the object before issuing a short-lived download capability.

SSE and long-polling consume connection capacity; configure heartbeat and proxy buffering policies. They should not share a worker pool with latency-sensitive requests without capacity planning.

## Failure, Caching, and Retries

A timeout may occur after a command commits. Do not automatically retry non-idempotent requests without an idempotency contract. Respect Retry-After for rate limits and service overload, apply jitter, and bound attempts and total time.

Cache failures only under an explicit policy. A stale success may be acceptable for a catalog; a cached authorization failure or payment result can be incorrect after state changes. Expose pending and degraded states rather than returning a plausible but false success.

## Testing and Operations

Measure warm and cold caches, representative payload sizes, slow clients, connection reuse, proxy buffering, compression, and dependency failure. Test conditional requests, private caching, pagination, duplicate commands, request deadlines, partial responses, and upload limits.

Monitor time to first byte, origin time, downstream time, response bytes, cache hit ratio, status classes, connection reuse, PHP-FPM queueing, and p95/p99 latency. Correlate a regression with deploy, payload growth, cache changes, proxy configuration, or dependency behavior. Load test with realistic keep-alive and concurrency rather than one benchmark client.

## Common Mistakes

- Measuring only PHP CPU time.
- Calling multiple providers serially without a deadline budget.
- Returning unbounded JSON arrays.
- Caching authenticated representations publicly.
- Assuming HTTP/2 or compression is end-to-end without measuring.
- Retrying commands after timeouts without idempotency.
- Streaming through a proxy that buffers the complete response.
- Serving large files through PHP workers unnecessarily.

## Senior Engineer Thinking

HTTP performance is an end-to-end budget. Reduce bytes and unnecessary work, reuse connections safely, cache with correct privacy and validators, parallelize only independent calls, queue or offload large work, and measure tail latency through the deployed proxy and client path.

## Exercises

1. Allocate a 1-second API budget across proxy, PHP, database, and two downstream providers.
2. Add ETag revalidation to a public representation and test private-cache behavior for an authenticated response.
3. Compare a paginated, compressed, and streamed export for a large dataset.
4. Design an outbound client deadline and retry policy for a command that may commit before a timeout.

## Review Questions

1. Which phases belong in an HTTP latency budget?
2. Why can connection reuse improve capacity?
3. What does an ETag validator do, and what does it not prove?
4. When should downstream calls be parallelized?
5. Why are large files often better served outside PHP workers?
6. Which HTTP retries require idempotency?

## Summary

HTTP performance includes client, connection, proxy, PHP, database, dependencies, serialization, and transfer. Set end-to-end budgets, reduce payloads, reuse connections safely, cache with privacy and validators, bound and parallelize downstream work deliberately, offload large transfers, and observe tail latency through the real deployment path.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [web.dev: Learn Performance](https://web.dev/learn/performance/)
- [PHP-FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)

