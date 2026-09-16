---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 163
title: End-to-End Tests
slug: end-to-end-tests
status: complete
summary: ../../_ai/chapter-summaries/163-end-to-end-tests-summary.md
---

# Chapter 163 — End-to-End Tests

## Why This Matters

An end-to-end test follows a user-visible workflow through the deployed application: browser, network, web server, PHP application, database, and selected external boundaries. It answers a question narrower tests cannot: can a real user complete this critical path in an environment close enough to production?

End-to-end tests are slower and more fragile than unit or API tests. Their value comes from covering integration seams that are otherwise easy to misconfigure. Keep the suite small, prioritize revenue, security, and recovery paths, and move detailed rule coverage to faster layers.

## Choose Critical Journeys

Describe a journey in user language and identify the observable outcome:

1. A user signs in with a test account.
2. The user creates a reservation.
3. The confirmation page shows the reservation and its status.
4. A second browser session cannot access another tenant's reservation.

The test should assert visible behavior and durable results, not private controller calls. A failed step should identify the URL, action, and expected state. Critical journeys include login and logout, purchase or booking, password reset, authorization boundaries, file upload, and a representative recovery path.

## Browser and Application Boundaries

A browser test may be driven by Playwright, Selenium, Symfony Panther, or another maintained tool. The language of the driver is less important than the boundary: use a real browser engine when JavaScript, cookies, redirects, storage, and rendering are part of the behavior.

A framework-neutral PHP test shape can keep scenario intent separate from the driver:

~~~php
<?php

declare(strict_types=1);

interface Browser
{
    public function visit(string $url): void;
    public function fill(string $field, string $value): void;
    public function click(string $selector): void;
    public function see(string $text): bool;
}

function userCanCreateReservation(Browser $browser): void
{
    $browser->visit('/login');
    $browser->fill('email', 'e2e-user@example.test');
    $browser->fill('password', 'test-only-password');
    $browser->click('[data-test=sign-in]');

    $browser->visit('/reservations/new');
    $browser->fill('court', 'court-7');
    $browser->fill('starts_at', '2026-09-20T18:00:00Z');
    $browser->fill('ends_at', '2026-09-20T19:00:00Z');
    $browser->click('[data-test=submit-reservation]');

    assert($browser->see('Reservation confirmed'));
}
~~~

The interface is illustrative. In a real suite, use stable semantic roles or dedicated data-test attributes rather than CSS classes tied to layout. Avoid selectors that depend on generated DOM structure.

## Deterministic Data and Isolation

Use a test environment with isolated credentials, database, object storage, email sink, and queues. Create data through an API or a fast fixture mechanism with unique names. Do not point end-to-end tests at production or real customer accounts.

Parallel workers need independent namespaces. A run identifier can prefix email addresses, tenant names, uploaded objects, and queue topics. Clean up in a finally block, and also provide a scheduled cleanup job because a killed browser process may skip teardown. Fixed dates and a controllable clock remove failures around midnight, daylight-saving changes, and token expiry.

Do not make every test register a new account through the public email flow. Seed verified accounts through a controlled test endpoint or database fixture, then reserve the real registration path for one or two dedicated journeys. This makes the suite faster without hiding a contract that matters.

## Waiting and Asynchronous Work

Wait for a condition that represents the intended state, not a fixed sleep. An end-to-end test can poll for a visible state or use a test-only queue drain, with a bounded timeout and useful diagnostics. The application should expose no production-only shortcut that bypasses authorization or business rules.

For asynchronous confirmation, assert the user-visible pending state first, then wait for the result. If a provider is not reliable enough for a deterministic suite, replace it with a local sandbox or controlled fake and keep a smaller provider smoke test. Capture browser console errors and failed network requests when a wait expires.

## Security and Failure Paths

Run tests with normal CSRF, authentication, authorization, cookie, and TLS policies for the environment. Test that an expired session returns to login, a wrong-tenant URL is rejected, a reset link is single-use, and a sensitive action requires step-up authentication. A successful login test alone does not prove session fixation resistance or logout revocation.

Never print passwords, session cookies, reset URLs, or personal test data in failure artifacts. Redact screenshots and traces when they may contain credentials or sensitive business data. Store artifacts with access controls and a retention period.

## Reliability and Parallelism

Flaky tests are tests whose result depends on timing, order, environment, or an uncontrolled external condition. A retry can make a dashboard green while hiding a real problem. If a runner retries, record the original failure and classify the cause; quarantine only with an owner and removal date.

Use one browser context per scenario where isolation is required. Reusing a logged-in context can save time, but it also hides login and cookie defects and lets state leak between tests. Choose parallelism based on database and browser capacity, not only CPU cores.

## Deployment and Diagnostics

Run end-to-end tests against a release candidate or a staging environment that matches production configuration. Verify migrations, assets, TLS termination, redirects, cookie domains, queues, and feature flags before the journey begins. A health check that only returns a process status is not enough to prove the journey's dependencies are ready.

On failure, preserve the URL, step name, screenshot or trace under a policy, browser console, network failure summary, server correlation ID, and selected server logs. Record the build, browser version, database schema, and test data namespace. Do not rerun automatically until the diagnostics are captured.

## Common Mistakes

- Turning every unit or validation rule into a slow browser journey.
- Using fixed sleeps instead of condition-based waits.
- Sharing accounts, cookies, or mutable data across parallel scenarios.
- Disabling security controls to make setup easier.
- Using brittle CSS selectors tied to visual layout.
- Retrying failures without preserving the original evidence.
- Letting real third-party email or payment traffic enter the suite.

## Senior Engineer Thinking

End-to-end tests are production-path probes. They should be few, independent, observable, and tied to business risk. Use them to find broken seams and deployment assumptions, then diagnose failures with the same correlation and release identity used in production.

## Exercises

1. Choose five critical journeys for a multi-tenant reservation application and assign each to an end-to-end or faster test layer.
2. Design a run namespace that isolates users, database rows, uploads, emails, and queue messages across ten parallel workers.
3. Replace a fixed sleep in an asynchronous test with a bounded condition wait and useful timeout diagnostics.
4. Define a redaction policy for browser traces and screenshots.

## Review Questions

1. What makes an end-to-end test different from an API test?
2. Why are fixed sleeps and shared browser state dangerous?
3. Which security controls should remain enabled in an end-to-end environment?
4. How can a retry hide a flaky or real defect?
5. What evidence should a failed browser journey preserve?

## Summary

End-to-end tests exercise a small set of critical user journeys through the browser and deployed application. Isolate data and credentials, wait on meaningful conditions, keep security controls active, control asynchronous dependencies, and preserve release and browser diagnostics. Their cost makes prioritization and failure classification essential.

## References

- [Playwright documentation](https://playwright.dev/docs/intro)
- [Selenium documentation](https://www.selenium.dev/documentation/)
- [Symfony Panther documentation](https://symfony.com/doc/current/testing.html)
- [PHPUnit documentation](https://docs.phpunit.de/)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

