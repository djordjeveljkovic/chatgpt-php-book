---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 216
title: Laravel Testing
slug: laravel-testing
status: complete
summary: ../../_ai/chapter-summaries/216-laravel-testing-summary.md
---

# Chapter 216 — Laravel Testing

## Why This Matters

Laravel testing helpers make it practical to exercise HTTP routes, validation, authentication, database interactions, jobs, events, mail, notifications, and console commands. The helpers reduce setup, but a passing framework assertion is not automatically proof that the application contract is correct.

Choose the smallest test boundary that answers the question. Unit tests prove domain rules quickly, feature or API tests prove Laravel routing and middleware composition, integration tests prove the real database and queue behavior, and browser tests prove a user journey. Exact class and helper names can change between Laravel majors; keep the test intent stable and check the target version's documentation.

## Feature and HTTP Tests

A Laravel feature test typically boots the application and sends a request through the router and middleware stack. Assert client-visible status, headers, and semantic JSON fields instead of private controller calls.

~~~php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use Tests\TestCase;

final class ReservationApiTest extends TestCase
{
    public function testCustomerCanCreateReservation(): void
    {
        $user = User::factory()->create();

        $response = $this
            ->actingAs($user, 'api')
            ->postJson('/api/reservations', [
                'court_id' => 'court-7',
                'starts_at' => '2026-09-20T18:00:00Z',
                'ends_at' => '2026-09-20T19:00:00Z',
            ]);

        $response
            ->assertCreated()
            ->assertJsonPath('data.status', 'confirmed');

        $this->assertDatabaseHas('reservations', [
            'user_id' => $user->id,
            'status' => 'confirmed',
        ]);
    }
}
~~~

The example assumes application models and factories exist. In a real suite, use the authentication guard and JSON shape defined by the application. Do not use a database assertion as a substitute for checking the response contract; assert both when both are requirements.

Test negative paths: unauthenticated and wrong-tenant access, validation, CSRF for browser mutations, unsupported media types, rate limits, stale ETags, idempotency conflicts, and provider failures. Assert the documented error status and problem fields. Keep credentials and personal fixture data out of failure output.

## Database Tests and Factories

Factories make fixture creation readable and let a test vary state deliberately. They should create valid domain data, but they must not bypass an invariant that production code is supposed to enforce. Use the framework's database reset trait or an explicit transaction/schema strategy according to how jobs and multiple connections behave.

Test casts, relationships, scopes, constraints, soft-delete policy, transactions, and optimistic conflicts against the production database engine where possible. An in-memory SQLite database can differ from PostgreSQL or MySQL in types, locking, SQL functions, and constraint behavior.

Avoid asserting every timestamp, generated ID, or framework metadata. Assert the state and relationships the requirement promises. For large fixtures, create only the rows needed and use bounded queries; a test that creates thousands of unrelated models obscures the scenario and slows the suite.

## Jobs, Events, Mail, and Notifications

Laravel provides fakes for jobs, events, mail, notifications, and storage. A fake can prove that the application submitted the expected message; it cannot prove that a worker serializes, retries, or delivers it correctly.

~~~php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Queue;

function testCheckoutDispatchesReceiptJob(): void
{
    Queue::fake();

    $response = testClient()->postJson('/api/checkout', [
        'cart_id' => 'cart-7',
        'idempotency_key' => 'test-key-1',
    ]);

    $response->assertAccepted();
    Queue::assertPushed(GenerateReceipt::class, function (GenerateReceipt $job): bool {
        return $job->cartId === 'cart-7';
    });
}
~~~

Use a real or compatible queue integration test for payload serialization, after-commit timing, visibility timeout, duplicate execution, and failed-job behavior. Test event listeners with the right transaction and authorization context. A fake that records a dispatch before commit can hide a production race.

## Container and HTTP Mocking

Laravel's container can bind a test implementation for an external client. Use a narrow interface or the framework's HTTP fake for deterministic provider responses. Assert meaningful request data and failure classification, not every incidental framework header.

~~~php
<?php

declare(strict_types=1);

interface FraudGateway
{
    public function check(string $paymentId): FraudDecision;
}

final class StubFraudGateway implements FraudGateway
{
    public function __construct(private FraudDecision $decision)
    {
    }

    public function check(string $paymentId): FraudDecision
    {
        return $this->decision;
    }
}
~~~

Bind the stub only in the test application context and keep at least one integration or contract test for the real adapter. Configure timeout, malformed response, rate limit, and ambiguous outcome cases; a success-only fake makes retry logic untested.

## Authentication, Authorization, and Security

Use test helpers to authenticate a known test user, then cover roles, tenants, resource ownership, revoked membership, session expiry, CSRF, mass assignment, and rate limits. A test user created in one tenant must not see another tenant's records through a changed URL, filter, export, job, or model scope.

Run security middleware in feature tests. If a test needs a bypass for setup, keep the bypass localized and preserve a suite that uses the production stack. Never place production credentials in environment files used by CI. Redact tokens, cookies, reset links, and response bodies containing personal data in test artifacts.

## Parallelism and Flaky Tests

Parallel test execution needs isolated database schemas, cache prefixes, files, queues, and external names. Framework reset traits may need configuration for each worker. Use unique fixture identifiers and deterministic clocks. Test order should not affect results; run randomized order and retain the seed when diagnosing a failure.

Do not hide an intermittent failure by increasing retries. Capture the first failure, browser or server logs, SQL and queue diagnostics, and environment identity. Move a synchronization rule to a deterministic lower-level test when a browser test is the wrong boundary.

## Test Organization and CI

Separate fast unit and feature suites from database, queue, browser, and provider suites. Tag or group tests by required services and run a useful fast gate on every change. Run the broader matrix against supported PHP, database, and Laravel versions in CI if compatibility is a product requirement.

Use factories, helpers, and traits to remove setup noise while keeping the scenario visible. A helper that silently logs in an administrator or disables middleware can make every test over-permissive. Review test utilities as production-like code: they can create invalid data or hide authorization defects.

## Diagnostics and Coverage

A failing test should report route, method, status, selected headers, redacted payload, database or queue context, and correlation ID. Laravel's exception output and dumped response should be bounded and safe for CI artifacts. Preserve the build and framework version.

Coverage is a signal about executed lines, not a proof of behavior. Require meaningful assertions at critical boundaries and use mutation or contract tests where line coverage can be misleading. Track test duration, flake rate, retries, and failure categories so the suite remains a delivery tool.

## Common Mistakes

- Testing controllers while bypassing routing and middleware.
- Using factories that create impossible or cross-tenant state.
- Assuming queue, event, or mail fakes prove delivery.
- Running only SQLite when production uses different database semantics.
- Disabling authorization and CSRF globally for convenience.
- Mocking every provider success and never testing timeouts or malformed responses.
- Sharing mutable fixtures or cache state across parallel workers.
- Treating coverage percentage as behavior evidence.

## Senior Engineer Thinking

Laravel's testing helpers are boundary tools. Use feature tests for the framework-composed HTTP path, real database and queue tests for infrastructure semantics, contract tests for external agreements, and browser tests for critical journeys. Keep test data and security controls realistic, make fakes explicit, and preserve diagnostics and version compatibility.

## Exercises

1. Write feature tests for reservation creation, invalid input, wrong-tenant access, and duplicate idempotency keys.
2. Add a database test for a conditional state transition and run it against the production engine.
3. Test a queued receipt job's after-commit dispatch and duplicate execution.
4. Build a CI grouping strategy for unit, feature, integration, browser, and provider tests.

## Review Questions

1. What does a Laravel feature test observe that a controller unit test does not?
2. Why do queue and mail fakes need integration tests around them?
3. Which database differences make SQLite an incomplete production substitute?
4. Why should security middleware remain enabled in feature tests?
5. How should parallel tests isolate mutable resources?
6. What does coverage fail to prove?

## Summary

Laravel testing helpers make HTTP, database, queue, event, and external-boundary tests practical, but they do not remove the need to choose the right test layer. Assert client contracts and authorization, use realistic factories and production-like database semantics, distinguish fakes from delivery tests, isolate parallel runs, preserve redacted diagnostics, and keep framework-version assumptions explicit.

## References

- [Laravel documentation: Testing](https://laravel.com/docs/testing)
- [Laravel documentation: HTTP tests](https://laravel.com/docs/http-tests)
- [Laravel documentation: Database testing](https://laravel.com/docs/database-testing)
- [Laravel documentation: Mocking](https://laravel.com/docs/mocking)
- [Laravel Dusk documentation](https://laravel.com/docs/dusk)
- [PHPUnit documentation](https://docs.phpunit.de/)

