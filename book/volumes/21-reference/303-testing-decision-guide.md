---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 303
title: Testing Decision Guide
slug: testing-decision-guide
status: complete
summary: ../../_ai/chapter-summaries/303-testing-decision-guide-summary.md
---

# Chapter 303 — Testing Decision Guide

## Why This Matters

Choose a test from the claim and the risk, not from a team’s preferred vocabulary. A passing unit suite can coexist with a broken SQL query, an authorization gap, a race, a queue retry storm, or an untested restore path.

## Map Claims to Evidence

Write the behavior first: “a user cannot reserve an occupied slot,” “a job is not applied twice,” or “a tenant cannot read another tenant’s result.” Then choose the cheapest evidence that exercises the boundary where the behavior can fail.

| Claim | Useful evidence |
| --- | --- |
| pure transformation | unit examples and properties |
| domain invariant | unit plus database/concurrency integration |
| SQL shape | real database integration and plan inspection |
| HTTP contract | contract and API tests |
| authorization | permission matrix and wrong-scope integration tests |
| queue delivery | duplicate, retry, poison, shutdown, and recovery tests |
| latency/capacity | representative load and saturation measurements |
| deployment recovery | migration, rollback, restore, or game-day evidence |

## Test Layers

Unit tests isolate deterministic policy and transformations. Integration tests exercise serialization, SQL, transactions, locks, cache behavior, and real adapters. Contract tests protect producer/consumer or client/provider assumptions. End-to-end tests prove a small number of user journeys across deployed boundaries; they are expensive and should not carry every assertion.

Property tests express invariants across generated inputs: normalization is idempotent, pagination has no duplicates under its defined snapshot, or a reservation conflict is never accepted. Example tests remain useful for named edge cases and readable failures.

Characterization tests record observable legacy behavior before a refactor. They should include output, ordering, errors, null/empty distinctions, and side effects. Recording behavior does not decide that every behavior deserves to survive; mark intended changes separately.

## Test Doubles and Real Boundaries

Mocks and stubs make a unit fast and deterministic, but they can hide SQL semantics, timeout phases, serialization, provider behavior, locking, and acknowledgement timing. Use a fake when a stable in-memory contract is sufficient; use a real dependency when the dependency’s behavior is the claim.

Avoid verifying only that a method was called. Assert the durable state, emitted message, authorization result, or user-visible contract. Keep test fixtures explicit about tenant, identity, clock, randomness, and feature flags.

## Time, Randomness, and Concurrency

Inject a clock or control time at the boundary. Test expiry, clock skew assumptions, retry deadlines, daylight-saving transitions, and long-running worker behavior. Seed or replace randomness when determinism is required, but retain a path that tests production entropy and failure handling.

Concurrency needs more than two sequential calls. Use database integration, barriers, isolation controls, or deterministic conflict injection to exercise overlap, duplicate delivery, stale versions, deadlocks, and retry behavior.

## Security and Tenant Isolation

Build a permission matrix for identities, actions, resources, and tenant scopes. Test direct object access, list queries, search, exports, cache hits, jobs, webhooks, CLI tools, repair scripts, and error responses. A controller test that hides a button is not proof of authorization.

## Performance and Recovery Evidence

Load tests should state workload, data distribution, cache state, concurrency, dependencies, target percentiles, saturation signals, and abort conditions. A microbenchmark can compare a local function; it cannot prove system capacity.

Recovery tests should inject failures at boundaries: after a database commit, before acknowledgement, after a provider request, during a migration, and while old/new workers coexist. Verify durable state, reconciliation, idempotency, alerting, and operator runbooks.

## Coverage and Test Health

Coverage highlights executed lines; it does not prove meaningful assertions or untested boundaries. Track mutation results, flaky-test rate, duration, quarantines, fixture age, production defect escape, and whether important claims have evidence. A green suite can be stale or too isolated.

## Test Decision Worksheet

| Question | Answer |
| --- | --- |
| What behavior or invariant is protected? | |
| At which boundary can it fail? | |
| What is the cheapest trustworthy test? | |
| Which real dependency must be exercised? | |
| What data, tenant, time, and version matrix is needed? | |
| What failure or unknown outcome must be injected? | |
| What production signal confirms the test’s assumption? | |

## Common Mistakes

- treating line coverage as quality;
- mocking every boundary;
- testing only the happy path;
- omitting wrong-tenant and duplicate-delivery cases;
- asserting incidental ordering;
- using real external services without bounded cost or isolation;
- ignoring time, randomness, retries, and concurrency;
- keeping characterization tests without a change decision;
- treating a test pass as deployment or restore evidence.

## Exercises

1. Map five system claims to trustworthy test layers.
2. Build a tenant permission matrix and derive negative tests.
3. Replace a provider mock with a contract test and failure injection.
4. Design a characterization fixture for a legacy import path.
5. Write a recovery test for a queue message acknowledged after an uncertain provider call.
6. Define load-test assumptions and guardrails for Chapter 287 search.

## Review Questions

- What determines the right test boundary?
- Why can a mock create false confidence?
- Which claims require real database or provider behavior?
- How do property tests complement examples?
- What does coverage fail to prove?
- How should recovery and unknown completion be tested?

## Summary

Testing decisions follow the behavior, invariant, boundary, and risk. Combine focused unit and property tests with selective real-boundary integration, contract, security, load, characterization, and recovery evidence. Measure test health and never confuse a green suite with system readiness.

## Chapter 304 Handoff

Once claims and risks are explicit, architecture decisions can choose proportionate boundaries. Chapter 304 provides an architecture decision guide for modules, services, queues, projections, caches, ownership, migration, and reversibility.

## References

- [Chapter 170 — Test Design](../11-testing/170-test-design.md)
- [Chapter 171 — Coverage](../11-testing/171-coverage.md)
- [Chapter 176 — Database Testing](../11-testing/176-database-testing.md)
- [Chapter 272 — Characterization](../18-legacy-php/272-characterization-tests.md)
- [Chapter 292 — Performance Investigation](../20-senior-engineering/292-performance-investigation.md)
- [Chapter 293 — Security Review](../20-senior-engineering/293-security-review.md)
