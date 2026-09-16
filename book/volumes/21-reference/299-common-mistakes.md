---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 299
title: Common Mistakes
slug: common-mistakes
status: complete
summary: ../../_ai/chapter-summaries/299-common-mistakes-summary.md
---

# Chapter 299 — Common Mistakes

## Why This Matters

Most production bugs are not caused by a missing language feature. They come from a plausible shortcut applied outside its assumptions: a truthiness check that erases a valid zero, a retry that duplicates a payment, or a query that authenticates a user but forgets tenant scope. Treat this chapter as a symptom-to-cause reference.

## How to Use the Catalog

For each suspected mistake, ask:

1. What behavior is observed?
2. Which invariant or contract was violated?
3. Which boundary allowed the mistake through?
4. What evidence would confirm the cause?
5. What is the smallest safe correction?
6. How will regression and recovery be verified?

Not every unusual choice is a mistake. A deliberate trade-off with an owner, evidence, and review condition is different from an accidental shortcut.

## Language and Value Semantics

### Confusing missing, null, false, empty, and zero

Truthiness is a control-flow convenience, not a domain model. `0`, `'0'`, `''`, `false`, `null`, an absent key, and an empty collection can carry different meanings. Define the allowed state first, then use an explicit comparison or a value object. Test boundaries such as a zero price, an empty name, and a missing optional field.

### Relying on implicit conversions

String/number coercion, array key behavior, date parsing, and loose equality can make invalid input appear valid. Normalize at the boundary, use strict comparisons where identity matters, and record rejected forms. A cast is not validation.

### Assuming a copy is always cheap

Copy-on-write can defer copying, but mutation, references, and PHP array overhead still matter. Do not “optimize” by sharing mutable state or by accumulating a large result when a stream would suffice. Measure representative memory, not only elapsed time.

### Treating warnings as harmless

Warnings, deprecations, exceptions, and fatal errors have different propagation and deployment consequences. Make the error policy explicit, capture actionable telemetry, and test the supported PHP versions from Chapter 298.

## Boundaries and Data

### Validating after using the value

Parsing, authorization, normalization, and persistence are separate steps. Do not interpolate unvalidated input into SQL, shell commands, paths, HTML, redirects, or serialized messages. Validate the shape and policy at the boundary, then encode for the sink.

### Trusting client-provided tenant or object identifiers

Authentication proves identity, not permission. Scope the resource lookup and authorization decision together; carry the scope into caches, jobs, exports, indexes, and repair tools. Test wrong-tenant access through every entry point.

### Returning more data than the contract requires

Convenient arrays and ORM entities often expose internal fields, tokens, or other tenants’ data. Define response and export contracts, select only necessary columns, and review logs and error payloads as data surfaces.

### Assuming ordering is incidental

Without an explicit order and tie-breaker, pagination and tests can become unstable. State whether ordering is contractual, use cursor semantics when needed, and test inserts or updates between pages.

## Errors and External Effects

### Catching everything and continuing

A broad catch can turn data corruption or authorization failures into a successful response. Catch at a boundary that can classify the error, preserve context, choose a safe response, and alert when the process cannot guarantee correctness.

### Retrying without an operation identity

A timeout does not prove that work did not complete. Before retrying, determine whether the effect is idempotent, query durable evidence, and reconcile unknown completion. A retry budget limits amplification; it does not create idempotency.

### Assuming a database transaction rolls back email or payment

Database atomicity ends at the database boundary. Use an outbox, idempotent provider operation, durable status, and reconciliation for external effects. Test crash points between each step.

### Acknowledging a queue message too early

Acknowledging before durable completion loses work; acknowledging after every transient error can create a retry storm. Define acknowledgement, retry, poison-message, and shutdown behavior explicitly.

## Dependencies and Database Access

### Trusting the local environment

`php -v`, a successful Composer install, or a passing unit test does not prove FPM, worker, extension, database, or artifact compatibility. Compare the runtime matrix and inspect the deployed artifact.

### Hiding SQL behind an abstraction

ORM and query-builder code can still issue an N+1 query, omit a predicate, choose a poor plan, or hold a transaction too long. Inspect generated SQL, cardinality, plans, locks, and representative data.

### Using application checks for concurrent invariants

“Check then insert” races under concurrent requests. Put durable uniqueness or conflict enforcement at the database boundary and make the application handle the resulting conflict safely.

### Adding an index without measuring write cost

An index can help one query while slowing writes, consuming memory, or failing to match the query shape. Compare plans and workload-level metrics before and after.

## Testing, Performance, and Operations

### Treating coverage as proof

Line coverage cannot establish authorization, concurrency, serialization, capacity, or recovery. Map claims to unit, integration, contract, security, load, and failure-injection evidence.

### Mocking every boundary

A mock can hide SQL semantics, timeouts, locking, serialization, and provider behavior. Keep focused unit tests, but exercise critical real boundaries selectively.

### Optimizing averages

Average latency can improve while p99 worsens for large tenants. Define the workload, segment cohorts, decompose latency, inspect saturation and queues, and verify correctness and cost.

### Increasing concurrency to fix a slow dependency

More FPM workers or queue consumers may exhaust database connections, increase lock contention, amplify provider load, and raise memory pressure. Capacity is a shared constraint.

### Calling a process health check readiness

“The worker is alive” does not establish that it can consume safely, reach required dependencies, or preserve its message contract. Readiness and business capability checks need explicit evidence.

## A Triage Table

| Symptom | Common mistaken conclusion | Better first evidence |
| --- | --- | --- |
| timeout after a write | nothing happened | durable state, provider audit, idempotency record |
| data from another tenant | login is broken | query, cache, job, export, and policy scope |
| rising p99 | PHP is slow | trace, SQL plan, locks, pool saturation, queue age |
| repeated queue failures | retry harder | message version, poison payload, dependency error |
| tests pass, deploy fails | deployment is random | artifact, SAPI, extension, config, and version inventory |
| migration rollback fails | command was incomplete | schema/data compatibility and forward-recovery plan |

## Exercises

1. Classify ten reported bugs as language, boundary, data, dependency, concurrency, testing, performance, or operations mistakes.
2. For each, name the violated invariant and the evidence needed.
3. Rewrite a truthiness check, a check-then-insert flow, and an unsafe retry plan.
4. Trace tenant scope through a request, cache, queue, export, and repair path.
5. Create a regression matrix for an unknown-completion incident.

## Review Questions

- Why is a cast not validation?
- What is the difference between authentication and authorization?
- Why does a transaction not cover an email provider?
- What evidence distinguishes a slow PHP function from a saturated database?
- When is a mock harmful?
- Why can more workers worsen latency?
- What makes a deliberate trade-off different from a mistake?

## Summary

Common mistakes arise when implicit values, hidden boundaries, optimistic concurrency, broad error handling, unbounded retries, abstractions, mocks, averages, or process-health assumptions replace explicit contracts and evidence. Diagnose the violated invariant, gather boundary-specific evidence, apply the smallest safe correction, and verify recovery and regression.

## Chapter 300 Handoff

Repeated mistakes often become system shapes that look normal. Chapter 300 examines those recurring shapes as common anti-patterns, including when a familiar pattern is being used outside its context.

## References

- [Chapter 5 — CLI versus Web PHP](../01-the-php-mental-model/005-cli-vs-web-php.md)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 223 — Performance Mental Model](../15-performance/223-performance-mental-model.md)
- [Chapter 280 — Tennis Reservation Service](../19-small-engineering-projects/280-tennis-reservation-service.md)
- [Chapter 287 — Search/Filtering Service](../19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 298 — PHP Version Matrix](298-php-version-matrix.md)
