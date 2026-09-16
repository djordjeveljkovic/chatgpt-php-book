# AI Summary — Chapter 230 — HTTP Performance

- Status: complete
- Volume: Volume 15 — PERFORMANCE
- Last updated: 2026-09-16

## Written material

HTTP performance is covered as an end-to-end budget across clients, connections, TLS, proxies, PHP, databases, dependencies, serialization, caching, streaming, uploads, and large transfers.

## Concepts already explained

Latency budget, connection reuse, response size, compression, ETag, private caching, downstream deadline, partial response, streaming, and time to first byte.

## Examples used

PHP JSON and streaming generators, conditional ETag responses, and outbound deadline guidance.

## Cross-references

- [Chapter 124 — HTTP](../../volumes/09-http-and-application-development/124-http.md)
- [Chapter 126 — Responses](../../volumes/09-http-and-application-development/126-responses.md)
- [Chapter 127 — Headers](../../volumes/09-http-and-application-development/127-headers.md)
- [Chapter 229 — Database Performance](../../volumes/15-performance/229-database-performance.md)

## Open threads

Continue with Chapter 231 on OPcache.

## Exact next section

Chapter 231 — OPcache: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; deployed proxy and browser measurements were not run.
