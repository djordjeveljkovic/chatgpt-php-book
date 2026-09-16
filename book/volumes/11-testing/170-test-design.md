---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 170
title: Test Design
slug: test-design
status: complete
summary: ../../_ai/chapter-summaries/170-test-design-summary.md
---

# Chapter 170 — Test Design

## Why This Matters

A test suite can contain hundreds of tests and still miss the behavior that matters. Test design is the work of choosing conditions, observations, and evidence so a failure exposes a meaningful defect. It starts from the domain risk and turns it into cases that are clear, deterministic, and maintainable.

Good test design reduces two costs at once: defects that reach users and time spent diagnosing a failure. It avoids both a tiny suite that gives false confidence and a huge suite that repeats implementation details.

## Start with the Contract

Write the behavior in terms of an actor, precondition, action, result, and side effect. A reservation service might promise that an interval is accepted only when it is ordered, within business hours, authorized for the tenant, and non-overlapping. Each condition suggests a distinct test.

Use a decision table when multiple conditions interact:

| Authorized | Interval valid | Slot free | Expected result |
| --- | --- | --- | --- |
| no | yes | yes | denied |
| yes | no | yes | validation error |
| yes | yes | no | conflict |
| yes | yes | yes | reservation created |

The table exposes combinations that a happy-path test hides. Do not generate every theoretical combination blindly; use risk and domain rules to identify representative combinations and interactions.

## Boundaries and Partitions

Partition input into equivalence classes that should behave alike, then test boundaries where behavior changes. For a quantity from 1 through 10, useful cases include 0, 1, 10, 11, a wrong type, and a missing value. For a date interval, test equal endpoints, one second on either side, timezone changes, and maximum duration.

Boundary cases are not only numeric. Include empty and whitespace strings, duplicate collection members, absent versus null fields, Unicode normalization, unknown enum values, stale versions, expired credentials, and a missing database row. For each case, assert both the public result and the absence of forbidden effects.

~~~php
<?php

declare(strict_types=1);

function quantityCases(): array
{
    return [
        'minimum minus one' => [0, false],
        'minimum' => [1, true],
        'maximum' => [10, true],
        'maximum plus one' => [11, false],
    ];
}
~~~

A PHPUnit data provider can turn this table into executable cases. Keep the case name meaningful so a failure explains which boundary was violated.

## State and Time

Many bugs are invalid transitions rather than invalid values. Model a resource as states and legal transitions: pending to approved, pending to rejected, but not approved back to pending without an explicit process. Test every important allowed transition and representative forbidden transitions.

Inject a clock for expiry and scheduling. Choose exact boundary semantics, such as expired when now is greater than or equal to expiresAt, and test both sides. Use a fake random source or assert only format and uniqueness properties; never make a test depend on a particular production secret.

## Properties and Invariants

An example test checks one scenario. A property states what must remain true across many inputs. Sorting should preserve the multiset and produce a nondecreasing sequence. A parser followed by serialization should preserve the supported representation. A retry with the same idempotency key should not create a second effect.

Properties are useful with generated data, but keep preconditions explicit. A property about an interval sorter must generate valid intervals or separately test rejection of invalid intervals. Property-based testing is covered later in the volume; the design principle applies to ordinary data providers too.

## Test Observations

Choose observations that survive safe refactoring:

* returned value, error type, and stable error code;
* persisted state and database constraint outcome;
* emitted event type and safe fields;
* authorization result and response status;
* number of durable effects, especially for retries;
* externally visible headers, cookies, and content type.

Avoid asserting private call order, generated identifiers, full timestamps, SQL formatting, or JSON key order unless those are part of the contract. When order matters, state why: a FIFO queue has an ordering contract; two independent audit events may not.

## Fixtures, Builders, and Scenario Names

A fixture should make the scenario obvious. Use builders for valid defaults, but override the fields relevant to the case. Put invalid values directly in the test so the reader sees the boundary. Name tests with condition and result, such as testRevokedMemberCannotExportTenantData.

Keep one primary reason for failure in each test. A test that simultaneously depends on authentication, three database rows, a provider sandbox, and a clock is hard to diagnose; split it across test levels and keep one feature test for the complete flow.

## Failure and Threat Analysis

* **Missing combination:** a decision table has an untested interaction. Review conditions and high-impact combinations.
* **Unclear oracle:** the test runs but does not define correctness. State the observable contract before arranging fixtures.
* **Brittle assertion:** incidental order, timestamp, or generated ID changes. Assert stable behavior.
* **Fixture pollution:** hidden defaults mask the condition. Keep relevant values in the test.
* **Time race:** wall-clock time crosses a boundary during execution. Inject a clock.
* **Overfitting:** tests describe today's implementation rather than the domain. Use public behavior and invariants.
* **Security blind spot:** only authenticated owner cases exist. Add anonymous, wrong-tenant, revoked, and replay scenarios.

## Review Test Suites

When reviewing a test, ask what defect it would catch and whether a safe implementation change would preserve it. Read a failure without opening production code: a good name, fixture, and assertion should reveal the violated contract. Delete duplicate tests only after confirming another test covers the same risk.

A test suite should evolve with incidents, new state transitions, schema changes, and threat models. Test design is not complete when every line is executed; it is complete when important behavior has credible, maintainable evidence.

## Exercises

1. Build a decision table for a refund operation with actor role, payment state, amount, and provider status.
2. List equivalence classes and boundary values for a paginated API page size.
3. Model a document's state machine and write tests for every legal transition plus representative illegal transitions.
4. Replace a brittle test that asserts a generated ID and timestamp with assertions about stable identity and ordering properties.

## Review Questions

1. What does a decision table reveal that a happy-path test can hide?
2. How do equivalence classes and boundary tests complement each other?
3. Why should a test inject time and randomness?
4. Which observations usually survive safe refactoring?
5. When is ordering part of the test contract?
6. What question should a reviewer ask of every test?

## Summary

Test design turns domain risks into explicit conditions, observations, and evidence. Use decision tables, equivalence classes, boundary values, state models, properties, and controlled time. Assert stable public behavior and durable effects, keep fixtures clear, and review tests by the defect they would catch.

## References

- [PHPUnit Documentation](https://docs.phpunit.de/)
- [PHPUnit Data Providers](https://docs.phpunit.de/en/11.5/writing-tests-for-phpunit.html#data-providers)
- [Martin Fowler: Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [ISO/IEC/IEEE 29119 Software Testing](https://www.iso.org/standard/81291.html)

