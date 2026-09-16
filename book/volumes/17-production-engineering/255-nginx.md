---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 255
title: Nginx
slug: nginx
status: complete
summary: ../../_ai/chapter-summaries/255-nginx-summary.md
---

# Chapter 255 — Nginx

## Why This Matters

Nginx commonly sits at the edge of a PHP application. It accepts connections, serves static files, terminates TLS or passes it through, applies limits, and forwards dynamic requests to PHP-FPM over FastCGI. Its queues, timeouts, buffering, and routing rules shape what users experience before PHP runs.

An Nginx configuration is application behavior. A wrong document root can expose files, a missing body limit can permit resource exhaustion, and a timeout mismatch can leave PHP workers busy after the client has gone away.

## Request Path

```text
client → DNS/TLS → Nginx → FastCGI socket → PHP-FPM → application
          │          │          │
        limits     buffers    worker pool
```

Nginx and PHP-FPM are separate processes with separate configuration and logs. A successful Nginx response does not prove the application committed a business effect; a 502 or 504 may represent a failure before PHP, during FastCGI communication, or after PHP started work.

## Static and Dynamic Routing

Serve only the intended public directory. Route a known front controller to PHP and reject direct access to source, configuration, dependency metadata, and hidden files. Keep uploads outside the executable document root where possible.

A conceptual route looks like:

```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}

location ~ \.php$ {
    try_files $fastcgi_script_name =404;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php/php-fpm.sock;
}
```

The exact directives and socket path depend on the distribution and release layout. Review the rendered configuration and test that a request cannot select an arbitrary script path. Keep the public root explicit rather than relying on a broad directory.

## Limits and Timeouts

Set limits at the edge before PHP allocates expensive resources:

* maximum request body and header sizes;
* connection and request rate limits;
* client body and header timeouts;
* FastCGI connect, read, and send timeouts;
* response buffering and maximum temporary-file usage;
* upload and static-file policies.

These limits should fit the application deadline and PHP-FPM policy. A generous Nginx timeout can hold a worker during a provider outage; a short timeout can disconnect a client while PHP continues an ambiguous side effect. See [Chapter 238 — Timeouts](../16-distributed-systems/238-timeouts.md).

## FastCGI Boundary

FastCGI transports request metadata and body to a PHP-FPM pool. The socket or TCP connection has capacity and failure modes. A full listen queue, unavailable socket, or exhausted FPM pool can produce errors without PHP application logs.

Forward only the headers and parameters the application expects. Do not trust client-supplied internal identity headers unless a trusted proxy strips and sets them. Preserve request IDs and trace context under a documented policy, and avoid forwarding credentials to unrelated upstreams.

Nginx buffering can protect a slow client from holding PHP resources in some configurations, but it consumes memory or temporary storage. Streaming endpoints need deliberate buffering and flush behavior; do not apply a generic response policy to server-sent events.

## TLS and Proxy Headers

When TLS terminates at Nginx, the application needs a trusted way to know the original scheme and host. Configure trusted proxy behavior narrowly. A client must not be able to make the application believe an untrusted request was HTTPS merely by sending a forwarding header.

Use secure cookies, correct redirect behavior, and a canonical host policy. TLS certificate rotation and protocol configuration belong in the deployment runbook and should be tested before expiry.

## Caching and Compression

HTTP caching headers are a contract. Set `Cache-Control`, `ETag`, and `Vary` according to authorization, representation, and freshness. Do not cache private responses in a shared proxy. Static assets can use immutable, content-addressed names; HTML and API responses need a deliberate policy.

Compression reduces transfer bytes but consumes CPU and can interact with content types, buffering, and side-channel concerns. Measure transfer time, CPU, response size, and client behavior. Do not compress already compressed data without evidence.

## A Safe Release Layout

An atomic release layout makes document-root and code changes easier to reason about:

```text
/srv/app/releases/2026-09-16-1200/public
/srv/app/releases/2026-09-16-1200/vendor
/srv/app/current → releases/2026-09-16-1200
```

Point Nginx and PHP-FPM at a deliberate release boundary or use a configuration strategy that handles the overlap. Reload gracefully, verify the active version, and keep old and new database/message contracts compatible during the transition.

## Logs and Observability

Correlate Nginx access logs with PHP request logs using a safe request ID. Record status, request duration, upstream connect and response times, bytes, route template where available, and release. Distinguish client disconnect, upstream timeout, FPM failure, and application error.

Keep labels and log fields bounded. Never log authorization headers, session cookies, complete sensitive bodies, or query strings containing secrets. Protect access logs because URLs and referrers can contain personal data.

## Security

Run Nginx with least privilege, restrict filesystem access, disable directory listing unless explicitly required, block source and secret files, set appropriate security headers, and validate upload paths. Keep the admin/status endpoint internal and authenticated. A reverse proxy is not a replacement for application authorization.

## Testing

Test static and dynamic routing, wrong-case paths, source-file access, upload limits, oversized headers and bodies, TLS and proxy headers, FastCGI socket failure, full FPM pool, slow upstreams, client disconnect, streaming responses, cache headers, compression, graceful reload, and rollback.

Render and validate the actual Nginx configuration in CI or a safe deployment stage. Exercise it with representative requests; a syntax-valid configuration can still route a private file or omit a required security header.

## Common Mistakes

* Serving the repository root instead of the public directory.
* Allowing arbitrary PHP script paths through a broad location rule.
* Mismatching Nginx, PHP-FPM, database, and upstream timeouts.
* Trusting arbitrary forwarding headers from clients.
* Caching authenticated or tenant-specific responses publicly.
* Applying buffering that breaks streaming endpoints.
* Diagnosing only PHP logs when the FastCGI boundary is failing.
* Reloading configuration without testing the active release and rollback.

## Senior Engineer Thinking

Ask which requests Nginx can reject cheaply, which resources it can protect, and which errors occur before PHP sees the request. The proxy is part of the application boundary: routing, limits, cache semantics, security headers, observability, and reload behavior must be reviewed with the PHP code.

## Exercises

1. Review a configuration and list every route that can reach PHP, static files, uploads, or internal status.
2. Align Nginx, FastCGI, PHP-FPM, database, and provider deadlines for one endpoint.
3. Test a full FPM pool and distinguish Nginx, FastCGI, and application evidence.
4. Design cache headers for a public asset, tenant API response, and authenticated profile page.

## Review Questions

* What work should Nginx reject before PHP runs?
* Why can a 504 leave a remote effect unknown?
* Which forwarding headers require a trusted proxy policy?
* Why can buffering harm a streaming endpoint?
* Which files should never be served from the public root?
* What makes a configuration operationally safe beyond syntax validity?

## Summary

Nginx is a production boundary that routes, limits, buffers, caches, and observes traffic before PHP-FPM. Keep the public root narrow, align FastCGI and application deadlines, validate proxy identity headers, design cache and streaming behavior explicitly, correlate proxy and PHP evidence, protect sensitive files and endpoints, and test the rendered configuration under failure and reload conditions.

## References

- [Nginx documentation](https://nginx.org/en/docs/)
- [Nginx FastCGI module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)
- [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md)
- [Chapter 238 — Timeouts](../16-distributed-systems/238-timeouts.md)
