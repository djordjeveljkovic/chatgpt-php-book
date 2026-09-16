---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 272
title: Characterization Tests
slug: characterization-tests
status: complete
summary: ../../_ai/chapter-summaries/272-characterization-tests-summary.md
---

# Chapter 272 — Characterization Tests

## Why This Matters

A legacy system often has behavior that nobody can fully describe. Customers depend on it, scripts consume it, and other services may have encoded its quirks. Before changing such a system, you need a way to detect meaningful drift.

A characterization test records what the system does today at a chosen boundary. It is not automatically a test of what the system should do. The current behavior may include a bug, an accidental ordering, a weak error message, or a side effect that must eventually change. The test gives the team a visible decision point instead of allowing an unnoticed change.

The useful question is not “how do we add tests to old code?” It is “which observed behavior must remain stable while we make one architectural change?” Chapter 271 maps the boundaries; this chapter captures evidence at one of them.

## Mental Model

Treat a characterization test as an observation pipeline:

~~~text
input and environment
        ↓
legacy entry point
        ↓
result + side effects + diagnostics
        ↓
normalization
        ↓
oracle and decision
~~~

The input includes more than function arguments. It may include configuration, current time, locale, filesystem contents, database rows, session state, process user, current directory, and the exact runtime. The output includes more than a return value: status, headers, rendered content, database writes, files, messages, logs, warnings, exit codes, and external calls may be part of the contract.

Capture only the boundary that matters for the change. A test that records the whole application for every request becomes expensive and noisy. A test that records only a scalar can miss a duplicate email or a changed authorization decision.

## Behavior Is Not Yet a Specification

Separate three statements:

| Statement | Meaning |
| --- | --- |
| Observed | The old path produced this result under these conditions. |
| Required | The business or security contract says this result must hold. |
| Desired | The team wants a different result in a future change. |

Characterization tests primarily record `Observed`. Promote a behavior to `Required` only after a product, security, data, or operational decision. If a test is expected to fail after a planned correction, document why and replace it with a test of the intended contract.

Do not delete a failing characterization test merely because the old behavior looks ugly. Classify the difference: intended improvement, compatibility break, test harness defect, environmental drift, or unknown. The classification is part of the migration evidence.

## Choose a Boundary

Good first boundaries are narrow and observable:

* a pure calculation called by a page or batch job;
* a command that receives a validated request and returns a response;
* a repository query with a stable fixture database;
* a file importer that emits records and error counts;
* a queue handler with recorded acknowledgments and effects;
* an HTTP adapter whose request and response can be safely replayed.

Avoid beginning with a boundary whose setup includes the whole production network, real customer data, irreversible provider calls, and an unstable clock. If the capability cannot be isolated, first add a recording adapter or a read-only mode.

For each boundary write a test charter:

~~~text
capability: create invoice
entry points: web request, admin CLI
inputs: customer, lines, currency, actor
state: fixture rows and configuration version
observable output: status, invoice shape, audit record
external effects: provider call, email command
known unknowns: database trigger and retry behavior
~~~

The charter prevents the test suite from silently changing scope while the harness grows.

## Test Oracles

An oracle decides whether an observation is acceptable. Choose the smallest oracle that protects the intended transition:

* exact equality for stable scalar results or status codes;
* structural equality for a response shape while ignoring harmless formatting;
* invariant checks for totals, authorization, uniqueness, or state transitions;
* effect checks for one message, one file, or one audit record;
* sequence checks when ordering is part of the contract;
* bounded statistical checks for nondeterministic timing or sampling;
* differential comparison between old and candidate implementations.

An oracle must state what it intentionally ignores. “Snapshot matches” is incomplete if timestamps, generated identifiers, whitespace, database row order, or provider request IDs are unstable.

## A Small Observation Harness

Use modern PHP to describe a result even when the code under test still uses legacy globals:

~~~php
<?php

declare(strict_types=1);

final readonly class Observation
{
    /**
     * @param list<string> $messages
     * @param list<string> $files
     */
    public function __construct(
        public mixed $value,
        public array $messages,
        public array $files,
        public string $stdout,
        public int $exitCode,
    ) {
        if ($exitCode < 0) {
            throw new InvalidArgumentException('Exit code cannot be negative');
        }
    }
}

function normalizeObservation(Observation $observation): array
{
    $messages = array_values(array_map(
        static fn (string $message): string => preg_replace(
            '/request-[a-f0-9-]+/',
            'request-<id>',
            $message,
        ) ?? $message,
        $observation->messages,
    ));

    sort($messages);
    $files = $observation->files;
    sort($files);

    return [
        'value' => $observation->value,
        'messages' => $messages,
        'files' => $files,
        'stdout' => $observation->stdout,
        'exit_code' => $observation->exitCode,
    ];
}
~~~

The harness makes side effects first-class. It also shows a normalization risk: sorting messages is valid only if order is not part of the contract. If two messages are required to occur in sequence, preserve the sequence and assert it. A normalization rule is a design decision, not a convenience filter.

The `mixed` value is deliberate at the boundary. A legacy result may be a scalar, array, object, false value, or warning-driven partial result. Normalize it only after deciding which shape matters. Do not make a weak legacy boundary look typed by casting away meaningful differences.

## Fixtures and State

Fixtures should make the precondition visible and repeatable. Record:

* the rows, files, sessions, configuration, and permissions required;
* the runtime, SAPI, timezone, locale, and extension assumptions;
* the process user and current directory when they matter;
* the state that must be reset between cases;
* the cleanup behavior if the legacy path fails halfway through.

Prefer the smallest fixture that triggers the behavior. A full production dump hides causality and increases privacy risk. For database fixtures, preserve relevant constraints, indexes, triggers, defaults, and transaction behavior; an array fake may not reproduce those effects.

Do not let one test depend on the side effects of another. Reset sessions, globals, static caches, temporary files, database rows, and recorded calls. Long-lived workers need an additional reset strategy because process-local state can survive many cases.

## Capture the Negative Paths

Success cases reveal only one slice of a legacy contract. Add cases for:

* missing, empty, malformed, or duplicate input;
* unauthorized and cross-tenant access;
* missing database rows and constraint failures;
* unavailable filesystem, mail, queue, or provider dependencies;
* timeout after an external operation may have completed;
* repeated delivery, interrupted batch, and worker restart;
* warnings, notices, non-zero exit codes, and partial output;
* old and new configuration or message versions.

Record whether the old behavior is safe, unsafe, or merely unknown. A characterization test can preserve an unsafe behavior long enough to make a controlled change, but it should not turn that behavior into a permanent approval. Security and data-boundary findings need a separate remediation decision.

## External Effects

Never characterize a real payment, email campaign, destructive command, or customer notification by replaying it against production. Use a recording adapter, provider sandbox, quarantined mailbox, transaction rollback where it truly covers the effect, or a dry-run contract.

The test should identify each effect with stable fields:

~~~text
operation: invoice-created
recipient: redacted test address
payload shape: invoice id, amount, currency
attempt: 1
outcome: accepted / rejected / unknown
ordering: after database commit
~~~

If the old system has no operation identity, the missing identity is part of the finding. Do not deduplicate a comparison merely by array order or a generated timestamp. Chapter 243 covers delivery contracts; characterization tests should expose the current ambiguity so a later change can resolve it.

## Golden Masters and Snapshots

Golden-master tests can be useful for large rendered output, exported files, or response envelopes. Make the snapshot reviewable:

1. name the scenario and input fixture;
2. normalize only fields proven irrelevant;
3. separate security-sensitive values from the stored artifact;
4. review changes as behavior decisions, not formatting noise;
5. record whether the snapshot represents observed or required behavior.

Snapshots are weak when they are huge, generated from unstable data, or updated automatically without review. Prefer focused assertions for authorization, money, state transitions, and side effects. A small golden master can protect a rendering boundary while a focused test protects the invariant underneath.

## Runtime and Environment Limits

When testing PHP 5 code, run the harness under the exact supported runtime and SAPI when possible. PHP 8 syntax checking or a modern test double cannot prove PHP 5 parsing, extension, coercion, warning, or lifecycle behavior. If the old runtime is unavailable, label the result as partial evidence and record the missing environment.

Keep the harness itself modern if that improves safety, but communicate across a narrow boundary: a subprocess, fixture file, command protocol, or adapter. Do not load PHP 5 source into a PHP 8 process merely to make the test runner convenient.

## Differential Comparison

Compare old and candidate paths only where effects are controlled:

~~~text
same fixture + same configuration
             ↓
        old path ──→ normalized observation A
             │
        new path ──→ normalized observation B
             ↓
       oracle + difference classification
~~~

Classify differences by field and consequence. A changed whitespace-only rendering may be harmless. A changed row order may break a consumer. A missing audit record, changed authorization result, or duplicate provider call is significant even if the main response matches.

Do not run two writers against the same live data just to compare them. Use read-only, isolated, shadow, or recorded paths. The comparison must have a recovery story as well as an assertion.

## Confidence and Coverage

Characterization coverage is not the same as line coverage. Ask whether the tests cover:

* every entry point that reaches the capability;
* meaningful input classes and boundary values;
* authorization and tenant contexts;
* success, failure, timeout, retry, and restart behavior;
* database, filesystem, queue, and provider effects;
* web and CLI/runtime differences;
* observations that support rollback and reconciliation.

Track unknowns explicitly. A green suite can mean “the captured cases still match,” not “the system is correct.” Confidence increases when observations are diverse, fixtures are minimal, negative paths are exercised, and the environment is representative.

## Common Mistakes

* Turning every observed bug into a permanent business rule.
* Asserting only the return value and ignoring side effects or diagnostics.
* Normalizing away ordering, authorization, encoding, or duplicate effects.
* Reusing a production database or provider for convenience.
* Building fixtures so large that no one knows which precondition matters.
* Running old source under a different PHP runtime and claiming compatibility.
* Automatically accepting every snapshot update.
* Comparing old and new writers against shared live state.
* Hiding warnings and notices that reveal a changed failure contract.
* Failing to reset globals, static caches, sessions, files, or worker state.
* Treating a green characterization suite as proof of correctness.
* Letting the harness become a second production application with no owner.

## Senior Engineer Thinking

The senior question is not “how many tests can we add?” It is “which behavior must remain observable while we change ownership, which behavior is unsafe and needs a deliberate correction, and what evidence would distinguish an application change from an environment defect?”

A good characterization suite is a temporary bridge that can become a permanent contract where the behavior is valuable. It names uncertainty, controls effects, preserves negative paths, and makes differences reviewable. When a test no longer protects a decision or a migration boundary, retire it deliberately rather than accumulating archaeology in the test suite.

## Exercises

1. Choose one capability from the Chapter 271 architecture map. Write a test charter with entry points, state, outputs, effects, and unknowns.
2. Design an `Observation` for a CLI importer. Decide which output, files, exit codes, warnings, and database effects belong in the oracle.
3. Create three normalization rules and justify why each ignored field cannot affect correctness. Add one field that must not be normalized.
4. Build a differential comparison for an old and candidate calculation using the same minimal fixtures. Classify one harmless and one dangerous difference.
5. Add negative cases for a missing row, unauthorized actor, provider timeout, duplicate message, and interrupted batch. State which results are observed and which are required.

## Review Questions

* What does a characterization test record, and what does it not prove?
* Why must input include environment and process state?
* How do observed, required, and desired behaviors differ?
* What makes an oracle appropriate for a legacy boundary?
* When is normalization safe, and when does it hide a contract?
* Why should external effects be recorded through adapters or sandboxes?
* What limitations remain when PHP 5 is unavailable?
* Why are snapshots insufficient for authorization and side-effect invariants?
* How should a team classify a difference between old and candidate behavior?
* What evidence increases confidence beyond a green test run?

## Summary

Characterization tests capture observed behavior at a chosen legacy boundary so architectural change becomes reviewable. Define a narrow charter, include environment and side effects, distinguish observed from required behavior, choose an explicit oracle, keep fixtures minimal, cover negative paths, control external effects, normalize only proven-irrelevant fields, and label runtime limitations. Differential comparisons and snapshots are useful when their authority, normalization, and recovery boundaries are explicit. A green characterization suite preserves evidence; it does not automatically certify correctness.

## References

- [Chapter 158 — Why Tests Exist](../11-testing/158-why-tests-exist.md)
- [Chapter 165 — Test Doubles](../11-testing/165-test-doubles.md)
- [Chapter 170 — Test Design](../11-testing/170-test-design.md)
- [Chapter 176 — Database Testing](../11-testing/176-database-testing.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 260 — Logging](../17-production-engineering/260-logging.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 262 — Tracing](../17-production-engineering/262-tracing.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](./271-legacy-architecture.md)
- [Chapter 273 — Safe Refactoring](./273-safe-refactoring.md)
