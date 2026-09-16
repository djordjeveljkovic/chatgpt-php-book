---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 162
title: API Tests
slug: api-tests
status: complete
summary: ../../_ai/chapter-summaries/162-api-tests-summary.md
---

# Chapter 162 — API Tests

## Why This Matters

An API test exercises the boundary that a client actually depends on: the HTTP method and URI, headers, authentication, request representation, status code, response representation, and failure semantics. It catches a broken route, serializer, middleware rule, or authorization check that a unit test of one class cannot see.

API tests should be fast enough to run on every change and precise enough to explain a contract failure. They usually call the application through an in-process HTTP kernel or a test server, while replacing only dependencies whose real behavior belongs to another test layer.

## Test the Contract at the Boundary

Start with a client-visible scenario. A successful request should assert the status, important headers, and semantic body fields. Avoid asserting incidental JSON whitespace, object key order, framework exception class names, or an exact generated identifier unless those are part of the contract.

~~~php
<?php

declare(strict_types=1);

final readonly class ApiResponse
{
    /** @param array<string, string> $headers */
    public function __construct(
        public int $status,
        public array $headers,
        public string $body,
    ) {
    }

    /** @return array<string, mixed> */
    public function json(): array
    {
        return json_decode($this->body, true, 512, JSON_THROW_ON_ERROR);
    }
}

interface ApiClient
{
    /** @param array<string, string> $headers */
    /** @param array<string, mixed> $body */
    public function request(
        string $method,
        string $path,
        array $headers = [],
        array $body = [],
    ): ApiResponse;
}
~~~

The client abstraction keeps test intent readable. Whether it calls a Symfony kernel, a PSR-7 handler, or an HTTP test server is a harness decision. The test still needs to prove that the production routing and middleware composition are exercised.

## A Scenario with Authentication and Validation

Use a known fixture account and send the same headers a real client sends. Test authorization by object and tenant, not just that one user can receive a 200 response.

~~~php
<?php

declare(strict_types=1);

function testCreateReservation(ApiClient $client): void
{
    $response = $client->request(
        'POST',
        '/v1/reservations',
        [
            'Authorization' => 'Bearer test-user-token',
            'Content-Type' => 'application/json',
            'Accept' => 'application/json',
        ],
        [
            'court_id' => 'court-7',
            'starts_at' => '2026-09-20T18:00:00Z',
            'ends_at' => '2026-09-20T19:00:00Z',
        ],
    );

    assert($response->status === 201);
    $body = $response->json();
    assert(is_string($body['id'] ?? null));
    assert($body['status'] === 'confirmed');
    assert($response->headers['Content-Type'] === 'application/json');
}
~~~

Use the assertion library's failure messages in a real test suite; the built-in assert is only a compact example. Seed the fixture through a repository or migration helper, not by coupling every API test to undocumented internal IDs.

The negative cases are often more valuable than the happy path:

- malformed JSON and an unsupported content type;
- missing, expired, or insufficient credentials;
- a valid resource owned by another tenant;
- invalid dates, state transitions, and unknown fields;
- duplicate idempotency keys with the same and different payloads;
- a missing resource where the visibility policy must avoid disclosure;
- oversized bodies and unsupported methods.

Assert the documented error shape and status. A client should be able to distinguish an authentication failure, validation error, conflict, rate limit, and server failure without parsing a human message.

## Headers, Caching, and Pagination

API tests should cover headers that alter behavior, including ETag validators, cache directives, Location, Retry-After, content type, and correlation IDs. Test conditional updates with a stale If-Match, and verify that a response containing a private representation cannot be stored by a shared cache.

For collections, assert stable ordering, filtering, maximum page size, cursor continuation, empty pages, and invalid cursors. Do not assert a page's complete fixture list when the contract only promises ordering and selected fields. A test that depends on incidental insertion order can pass locally and fail after a database engine or fixture change.

## Isolation and External Services

Each test must control data it relies on. Transactions rolled back after each test are fast, but they do not isolate code that opens another connection or queues work asynchronously. Truncate or recreate the relevant schema when the test boundary requires it. Parallel workers need separate database schemas or namespaces and unique external resource names.

Replace payment, email, search, and partner HTTP calls with a test boundary whose behavior is asserted elsewhere. An API test should verify that the application maps a provider result to its public contract; a provider integration test should verify the HTTP client separately. Record the provider request when it is part of the behavior, but avoid asserting every private header.

## Concurrency and Recovery

API clients retry when a response is lost. Test the idempotency and conditional-request contract by sending the same command twice, concurrent commands where a unique constraint matters, and a timeout after a durable commit. Test 429 with a retry hint and verify that expensive work is rejected before it starts.

A test server that returns an immediate response cannot prove that a real reverse proxy preserves streaming, timeout, or cache behavior. Keep those concerns in an integration or deployment test while API tests cover the application-level response contract.

## Test Data and Security

Use test credentials that cannot reach production and scrub request/response logs. Include authorization regression cases for every tenant and role introduced by a feature. Never make a test pass by disabling CSRF, validation, or authentication middleware globally; if a test needs a controlled bypass, make it explicit and reserve at least one suite for the production middleware stack.

Property-based input generation can find parser and validation edges such as Unicode, numeric boundaries, duplicate JSON fields, and long strings. Add a small regression fixture when a generated case reveals a defect.

## Diagnostics and CI

A failing API test should report method, path, status, selected headers, a redacted request, and a bounded response body. Preserve server logs and correlation IDs for failures, but remove access tokens and personal data. Run fast API tests on every commit and a broader matrix against supported PHP and database versions in CI.

Separate contract failures from environment failures. A service unavailable error from a test dependency should not be reported as an application assertion failure. Health checks and explicit dependency diagnostics make retries and triage more reliable.

## Common Mistakes

- Testing only controller methods and never routing or middleware.
- Asserting the entire JSON body when only a few fields are contractual.
- Sharing mutable fixtures between tests.
- Disabling security middleware so every test becomes easy.
- Hiding real provider calls in a suite that should be deterministic.
- Ignoring duplicate requests, stale validators, or tenant boundaries.
- Treating a test server response as proof about every production proxy.

## Senior Engineer Thinking

An API test is an executable client contract at the application boundary. Keep it broad enough to include routing, serialization, authentication, authorization, and persistence behavior, but narrow enough to isolate external systems and diagnose failures. The test suite should make compatibility and recovery behavior visible, not merely count successful status codes.

## Exercises

1. Write scenarios for a reservation API covering creation, validation, wrong-tenant access, duplicate idempotency keys, and stale ETags.
2. Design fixtures that can run in parallel without sharing mutable database rows.
3. Add a diagnostic formatter that redacts Authorization headers and limits response bodies.
4. Decide which cache, proxy, and provider behaviors require a separate integration test.

## Review Questions

1. Which parts of an HTTP API contract should a test assert?
2. Why are negative authorization cases essential?
3. What can a fast in-process API test fail to prove about a production proxy?
4. How should API tests handle external payment or email services?
5. Why can exact full-body assertions make a suite brittle?

## Summary

API tests exercise a client-visible HTTP boundary through routing, middleware, serialization, and application behavior. Assert status, important headers, semantic fields, authentication and authorization failures, validation, pagination, caching, idempotency, and recovery. Control data and external services deliberately, preserve diagnostics, and keep proxy or browser behavior in the test layer that can observe it.

## References

- [PHPUnit documentation](https://docs.phpunit.de/)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [PSR-7: HTTP Message Interface](https://www.php-fig.org/psr/psr-7/)
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)

