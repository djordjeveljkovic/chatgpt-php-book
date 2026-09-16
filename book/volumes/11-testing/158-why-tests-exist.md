---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 158
title: Why Tests Exist
slug: why-tests-exist
status: complete
summary: ../../_ai/chapter-summaries/158-why-tests-exist-summary.md
---

# Chapter 158 — Why Tests Exist

## Why This Matters

A test is an executable claim about behavior. It tells a future reader what the system promises, detects a regression when code changes, and makes a failure reproducible. Tests are useful because software is changed more often than it is first written: requirements evolve, dependencies are upgraded, schemas migrate, and production incidents reveal assumptions no one had documented.

Tests do not prove that a system has no bugs. They provide evidence about selected behavior under selected conditions. Good tests make important behavior cheap to check, failures easy to localize, and risky changes safer to review.

## Start with Risk and Behavior

Test a behavior a user, another service, an operator, or a business rule depends on. A private method name is usually a poor test target because it can change while the behavior remains stable. A reservation cannot overlap another reservation, an unauthorized user cannot read a tenant's invoice, and a retry must not charge twice are behaviors worth preserving.

Choose tests from risk:

* **Impact:** what happens if this behavior is wrong?
* **Likelihood:** how easily can it regress?
* **Observability:** how quickly would production reveal the mistake?
* **Cost:** how expensive is a test to write and maintain?

A payment invariant deserves stronger evidence than a formatting preference. A test that merely repeats an implementation line by line gives confidence without detecting meaningful change.

## The Test Contract

A useful test has a clear arrange, act, and assert shape. Its fixture names the relevant state, its action is one meaningful operation, and its assertions describe the observable result. Keep unrelated setup out of the test so a failure points toward one cause.

~~~php
<?php

declare(strict_types=1);

function testExpiredInvitationCannotBeAccepted(): void
{
    $clock = new FrozenClock(new DateTimeImmutable('2026-09-16T12:00:00Z'));
    $invitation = new Invitation(
        email: 'reader@example.test',
        expiresAt: new DateTimeImmutable('2026-09-16T11:59:59Z'),
    );
    $service = new InvitationService($clock);

    $result = $service->accept($invitation);

    assert($result === InvitationResult::Expired);
}
~~~

The example states the domain behavior. It does not assert which internal branch ran or whether a particular helper was called. Use a test framework's assertions in a real suite; the key design is that the test names the condition and expected effect.

## Tests as Documentation and Design Feedback

A test explains how a component is meant to be used. If constructing a service requires a database, clock, network client, queue, and global configuration for a simple rule, the setup is feedback about coupling. A small, explicit dependency can make both production behavior and tests easier to reason about.

Tests expose ambiguity. Before writing a test, decide whether an absent field differs from null, whether a retry returns the original response, whether authorization failure is 403 or an intentionally hidden 404, and whether time is measured at request start or commit. The resulting test becomes a precise answer for future maintainers.

Avoid tests that encode accidental details: exact generated IDs, array order where the contract has no order, random salts, timestamps without a clock, or full HTML snapshots when one accessibility label matters. Assert the contract at the right level.

## Test Levels and Feedback

No single test level covers the whole system:

* **Unit tests** isolate a small decision and run quickly.
* **Integration tests** verify collaboration with a real database, filesystem, queue, or provider boundary.
* **Feature tests** exercise an application flow through routing, middleware, serialization, and persistence.
* **End-to-end tests** drive a deployed system through real clients and infrastructure.
* **Contract tests** check an agreement between independently deployed parties.

Fast tests provide tight feedback; broader tests provide confidence about wiring and real behavior. The useful distribution depends on the system's risks. Do not chase a numeric pyramid ratio while leaving critical integration paths untested.

## Determinism and Isolation

A reliable test controls sources of variation: time, randomness, environment, network, database state, process order, and concurrency. Use a fake clock, injectable random source, temporary directories, isolated database transactions or schemas, and explicit fixtures. Parallel tests must not share mutable keys or ports.

Isolation does not mean hiding every integration. It means one test's result should not depend on another test's order or leftover state. When a test requires a shared service, provision a clean namespace and make cleanup failure visible.

## What Tests Cannot Replace

Tests do not replace authorization design, constraints, monitoring, review, backups, or incident response. A passing suite can miss an untested route, a misconfigured production proxy, a dependency compromise, or a race that the test scheduler never triggered. Use tests together with static analysis, security review, observability, and operational exercises.

Coverage reports show which lines or branches ran; they do not show whether assertions checked the right behavior. A suite can execute every line while accepting an incorrect result. Measure coverage trends and untested risk, not coverage as a goal by itself.

## Failure and Threat Analysis

* **Brittle tests:** assert private implementation details and fail during safe refactors. Assert stable behavior.
* **False confidence:** tests use mocks that cannot reproduce database constraints or serialization. Add integration evidence.
* **Flaky tests:** depend on time, order, network, or shared state. Control variation and isolate resources.
* **Slow feedback:** every test starts the whole stack. Keep a fast unit path and schedule broader suites appropriately.
* **Missing negative cases:** only happy paths are tested. Include invalid, unauthorized, duplicate, timeout, and concurrent behavior.
* **Unreviewed fixtures:** sensitive production data enters tests. Use synthetic or scrubbed data and protect test secrets.

## Review and Maintenance

Review tests like production code. Remove duplicate cases, name the business condition, keep failures diagnostic, and update tests when the contract changes. A failing test may identify a product change rather than a code defect; decide deliberately whether the behavior or the test should change.

When a production incident occurs, add a regression test at the lowest level that reproduces the failure and a broader test proving the relevant integration path. Keep the test focused on the failure's contract so it remains useful after implementation changes.

## Exercises

1. Choose one high-impact operation and list its success, invalid, unauthorized, duplicate, timeout, and concurrent behaviors.
2. Rewrite a test that asserts a private method so it asserts the public behavior instead.
3. Find three sources of nondeterminism in a PHP test suite and introduce a controllable clock, random source, or isolated resource.
4. Build a test-level map for a feature and identify one risk each unit, integration, feature, and end-to-end test would cover.

## Review Questions

1. What makes a test an executable specification?
2. Why is a private implementation detail usually a poor test target?
3. How do unit and integration tests provide different evidence?
4. What sources of nondeterminism commonly create flaky PHP tests?
5. Why does line coverage not prove a behavior is correct?
6. How should a production incident change the test suite?

## Summary

Tests provide executable evidence about behavior under chosen conditions. Start from risk and public contracts, keep tests deterministic and isolated, use multiple levels for different boundaries, and assert meaningful outcomes rather than implementation details. Tests work with static analysis, constraints, monitoring, review, and recovery; they do not replace them.

## References

- [PHPUnit Documentation](https://docs.phpunit.de/)
- [PHP Manual: Assertions](https://www.php.net/manual/en/function.assert.php)
- [Martin Fowler: Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

