---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 161
title: Feature Tests
slug: feature-tests
status: complete
summary: ../../_ai/chapter-summaries/161-feature-tests-summary.md
---

# Chapter 161 — Feature Tests

## Why This Matters

A feature test exercises a user-visible application flow across several layers: routing, middleware, authentication, authorization, validation, domain services, persistence, and response serialization. It catches wiring errors that unit tests and isolated adapter tests cannot see.

A feature test should represent a meaningful use case, such as signing in and creating an order, rejecting a cross-tenant request, or returning a validation problem. It should stop at the application boundary; driving a real browser and deployed infrastructure belongs to end-to-end testing.

## Define the Scenario

Start with an actor, precondition, request, expected response, and state change. Include the negative path that matters to the risk. A test name should describe a business outcome, not a controller method.

~~~php
<?php

declare(strict_types=1);

final class CreateOrderFeatureTest extends WebTestCase
{
    public function testAuthenticatedCustomerCanCreateAnOrder(): void
    {
        $customer = $this->fixtures->customer();
        $this->actingAs($customer);

        $response = $this->postJson('/orders', [
            'items' => [
                ['sku' => 'book-1', 'quantity' => 2],
            ],
        ]);

        self::assertSame(201, $response->status());
        self::assertSame('pending', $response->json('status'));
        self::assertDatabaseHas('orders', [
            'customer_id' => $customer->id,
            'status' => 'pending',
        ]);
    }
}
~~~

WebTestCase, actingAs, postJson, and database assertions are framework-shaped pseudocode. The contract is the important part: a real authenticated request returns a creation status and causes the expected persistence effect.

## Exercise the Real Boundary

Feature tests should use the application's router and middleware. They should prove content-type handling, authentication, authorization, CSRF behavior where applicable, validation errors, status codes, cookies, redirects, and response shape. Avoid calling a controller method directly; doing so skips the boundary where many bugs occur.

Keep external providers controlled. A feature test can use a provider fake or a local test server while still exercising the application's HTTP and queue wiring. Assert that the correct command or outbox record is created, then cover provider protocol behavior in integration tests.

## Authentication and Authorization Cases

Create explicit actors for anonymous, authenticated, wrong-tenant, disabled, and privileged cases. Test object-level authorization, not only role labels. A request that changes an ID must not access another tenant's resource, and a hidden resource should produce the documented 404 or 403 policy consistently.

Do not reuse an authenticated session accidentally between tests. Rotate or reset state as production does, and assert cookie attributes and logout behavior where sessions are part of the feature. For APIs, send real authorization headers through the middleware and test expired or malformed credentials.

## Validation and Error Contracts

Assert status, content type, safe problem shape, field errors, and absence of side effects. A validation response should not contain stack traces, SQL, credentials, or unbounded input echoes. Include malformed JSON, unknown fields, missing values, invalid relationships, oversized payloads, and duplicate commands.

Test one representative success path and the high-risk failures. Do not create a giant test that verifies every field, header, and database row for every scenario; it becomes difficult to update and failures become noisy. Use focused tests for separate contracts.

## State and Asynchronous Work

A feature may enqueue work rather than complete it synchronously. Assert the durable command or outbox event and its identifiers, then test worker behavior separately. If the endpoint promises a visible status transition, poll through a bounded test helper or use a deterministic worker trigger; never add arbitrary sleeps.

Use a real database transaction strategy appropriate to the application. A test wrapper that rolls back only the request connection may not cover a queue worker or an independently committed connection. Reset queues and idempotency records between scenarios.

## Time, Randomness, and External Effects

Inject a clock or freeze time at the test boundary when expiry, scheduling, or timestamps matter. Control randomness when asserting a token's format or lookup; never assert a specific production secret. Use fake payment, email, storage, and message providers and assert the meaningful call or durable result.

A feature test can verify redaction by capturing application logs, but do not print full request or response payloads when a test fails. Test tracing and correlation headers where clients or operators rely on them.

## Failure and Threat Analysis

* **Bypassed middleware:** direct controller calls miss authentication or content negotiation. Send a real application request.
* **Leaked test state:** shared sessions, databases, or queues make scenarios order-dependent. Isolate them.
* **Over-specified response:** asserting incidental JSON ordering or generated IDs makes safe changes fail. Assert the contract.
* **Arbitrary sleeps:** asynchronous tests become slow and flaky. Use a fake clock, deterministic worker, or bounded polling.
* **Provider leakage:** live email, payment, or storage calls affect real systems. Use controlled providers and dedicated credentials.
* **Authorization gaps:** only happy-path owner tests exist. Include anonymous, wrong-tenant, revoked, and privileged cases.
* **Unbounded input:** test fixtures create unrealistic payloads. Include size and count limits without flooding CI.

## Organizing the Suite

Group feature tests by user capability or bounded context rather than controller file. Keep fixture builders close to the domain and name scenarios clearly. Run a fast feature subset on pull requests and the broader suite in CI according to execution time and risk.

When a feature test fails, first classify the layer: route, middleware, serializer, domain, database, or external adapter. The test should give enough identifiers and captured diagnostics to locate the failure without logging secrets.

## Exercises

1. Write feature tests for create-order success, anonymous access, wrong-tenant access, invalid items, duplicate idempotency key, and provider timeout.
2. Add a test that proves a validation error creates no order and publishes no event.
3. Test logout and a copied session ID through the real request boundary.
4. Replace an arbitrary sleep in an asynchronous test with a deterministic worker trigger or bounded polling helper.

## Review Questions

1. What does a feature test cover that a controller unit test misses?
2. Why should feature tests use the router and middleware?
3. Which authentication and tenant cases belong in a feature suite?
4. How should asynchronous side effects be tested without sleeps?
5. Why should generated IDs and JSON key ordering rarely be exact assertions?
6. What must a feature test do to prevent live provider effects?

## Summary

Feature tests exercise meaningful application flows through routing, middleware, authentication, authorization, validation, persistence, and response mapping. Use real requests with controlled external providers, isolate sessions and data, assert public contracts and durable effects, cover high-risk negative paths, and make asynchronous tests deterministic.

## References

- [PHPUnit Documentation](https://docs.phpunit.de/)
- [PHPUnit Assertions](https://docs.phpunit.de/en/11.5/assertions.html)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Martin Fowler: Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)

