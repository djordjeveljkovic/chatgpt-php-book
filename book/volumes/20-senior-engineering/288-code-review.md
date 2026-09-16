---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 288
title: Code Review
slug: code-review
status: complete
summary: ../../_ai/chapter-summaries/288-code-review-summary.md
---

# Chapter 288 — Code Review

## Why This Matters

Code review is a risk-assessment activity performed with incomplete information. A reviewer is not certifying that a diff is attractive or that every line is understood. The reviewer is asking whether the change preserves the right behavior, enforces the right boundaries, and can be operated and recovered after release.

A small PHP diff can change SQL cardinality, authorization scope, serialized messages, transaction timing, memory use, or rollback safety. Review quality comes from asking the right questions about the change’s contract and evidence, not from producing the most comments.

## Define the Review’s Purpose

Before reading the diff, identify:

~~~text
change: add tenant-scoped product search
owner: catalog team
user-visible contract: filters, ordering, cursor, and response fields
invariants: tenant isolation, stable order, bounded query cost
evidence: parser tests, real SQL tests, plan review, load sample
operational change: new indexes and cache keys
rollback: disable route/cache; retain compatible cursors and data
review decision: approve, request changes, or record accepted risk
~~~

The review depth should match risk. A typo in documentation needs a different review from a payment mutation, schema constraint, authentication policy, or queue consumer. Small line count does not imply small blast radius.

## Review Context Before Details

Read the issue, design note, API or domain contract, migration, tests, configuration, and deployment plan before judging individual lines. Ask what is intentionally changing and what must remain unchanged. A diff can be locally correct while violating an unstated contract owned by another service or operator.

Build a change map:

| Surface | Review question |
| --- | --- |
| inputs | what is trusted, parsed, bounded, and rejected? |
| decisions | which business rules or authorization policies changed? |
| durable state | which rows, schemas, caches, files, or messages change? |
| effects | can email, payment, webhook, queue, or notification duplicate? |
| dependencies | what provider, extension, database, or runtime behavior is assumed? |
| operations | what metrics, alerts, capacity, rollout, and rollback change? |
| ownership | who handles failures and removes temporary compatibility code? |

Review the code that calls the changed code, not only the changed method. A return-value change, new exception, changed default, or altered transaction boundary may affect callers outside the diff.

## Start with Correctness and Invariants

State the invariant in plain language, then trace every path that can violate it. For a reservation, it might be “no two active reservations overlap on one court.” For a search service, it might be “every result belongs to the authenticated tenant and follows one total order.”

Check:

* normal, empty, malformed, duplicate, and retry paths;
* boundaries such as zero, null, expiry, time zones, and maximum sizes;
* concurrent requests and interleavings;
* partial commits and ambiguous completion;
* old and new versions during deployment;
* failure and recovery behavior.

Tests are evidence, not the invariant itself. A test that passes on a mocked repository does not prove a database lock, unique constraint, provider contract, or cache eviction behavior.

## Review Security and Authorization

Follow data across trust boundaries. Ask:

* Is identity derived from authenticated server state?
* Is tenant or object authorization applied in every read, write, export, cache, job, and repair path?
* Are SQL identifiers allow-listed and values parameterized?
* Are URLs, files, commands, templates, headers, and serialized values validated for their actual sink?
* Are secrets and personal data absent from logs, metrics, exceptions, and queue payloads?
* Does a new endpoint alter enumeration, timing, quota, or abuse behavior?

Avoid approving a “security fix” that merely moves the check. If authorization occurs after data was loaded from another tenant, the query and cache boundary are already wrong. If a new filter lowers authorization in PHP after an unbounded query, it may create both a leak and a denial-of-service path.

## Review Data and Transaction Boundaries

A migration, upsert, cache invalidation, or outbox insert can be more important than the application method around it. Inspect:

* schema constraints and existing invalid data;
* transaction start, commit, rollback, and isolation;
* lock scope and canonical lock order;
* affected rows and retry behavior;
* backfill cursor and concurrent updates;
* outbox/inbox or idempotency identity;
* backup, restore, forward-recovery, and rollback compatibility.

Do not accept “the code is in a transaction” as a complete argument. A transaction that calls a slow provider holds resources, while a transaction that omits a required write can leave a publish gap. Review what becomes durable before an acknowledgement or external effect.

## Review Performance and Capacity

Ask what workload changed:

* requests per second, queue rate, batch size, and concurrency;
* rows examined, memory retained, response bytes, and provider calls;
* database connections, locks, indexes, cache keys, and network fan-out;
* hot keys, noisy tenants, and retry amplification;
* startup, worker recycling, and deployment drain behavior.

A new index may improve one query while slowing imports. A cache may lower average latency while causing a stampede on expiry. A faster PHP function may increase downstream concurrency. Request measurements, a capacity estimate, and a rollback trigger appropriate to the risk.

## Review Tests as Evidence

Classify each important claim:

~~~text
static: parser, types, and code shape
unit: deterministic local rule
integration: database/cache/queue/provider boundary
contract: external message or API semantics
load: capacity, contention, and resource behavior
deployment: mixed versions, rollout, and rollback
recovery: crash, timeout, replay, restore, and repair
~~~

Look for missing negative and failure cases rather than counting assertions. A change that adds a retry needs an ambiguous-outcome test. A change that adds a cache needs invalidation and tenant-isolation tests. A change that adds a migration needs old-reader/new-writer compatibility and restore evidence.

Coverage can show that lines ran; it cannot show that an oracle is correct or that a production database plan is acceptable. Ask what evidence would falsify the author’s safety claim.

## Make Review Findings Actionable

Good findings identify consequence, location, and a path to resolution:

~~~text
severity: high
location: reservation insert path
observation: overlap is checked before insert without shared serialization
consequence: concurrent requests can create two active reservations
evidence needed: database constraint or lock-row transaction test
suggestion: move check and insert under the agreed court lock
~~~

Separate blocking findings from suggestions. A naming preference is not equivalent to a tenant-isolation failure. Explain the contract and risk, not the reviewer’s personal implementation preference. If the issue is uncertain, request a focused experiment or evidence instead of presenting speculation as fact.

Use severity consistently, and keep clarification requests separate from severity:

| Level | Meaning |
| --- | --- |
| blocker | release would violate safety, security, data, or required contract |
| high | likely material failure or operational risk; resolve before merge |
| medium | correctness, maintainability, or evidence gap with bounded impact |
| low | useful improvement that need not block this change |
| question | clarification needed before confidence is possible; classify the underlying risk separately |

## Review the Diff for Shape and Ownership

Look for changes that signal hidden scope:

* new global state, singleton caches, or static registries;
* widened interfaces and default behavior changes;
* catch-all exception handling or silent fallback;
* generated code or dependency updates without lock/provenance evidence;
* migrations mixed with unrelated business changes;
* feature flags without expiry, owner, or metric;
* logging or metrics with unbounded dimensions;
* compatibility adapters without removal criteria.

A clean abstraction can still hide a wrong owner. Ask which layer owns authorization, transaction, retry, idempotency, cache invalidation, and external effects. The class with the shortest method is not necessarily the correct authority.

## Review Workflow and Automation

Automate repeatable checks: formatting, syntax, static analysis, dependency policy, tests, secret scanning, migration checks, link checks, and artifact reproducibility. Automation narrows the surface a human must inspect; it does not replace judgment about business invariants, threat models, capacity, or recovery.

Keep pull requests reviewable. Separate mechanical renames from behavior changes, include a migration and rollout note, link evidence to claims, and explain generated files. A reviewer should be able to reconstruct why the change is safe without reverse-engineering an entire history.

For high-risk changes, use more than one perspective: domain owner, security reviewer, database reviewer, operations owner, or incident/recovery partner. Avoid approval theater where many people click approve without a clear decision owner.

## Resolve Disagreement

Disagreement is useful when it exposes an unstated contract. Make the disagreement concrete:

1. state the competing claims;
2. identify the invariant or decision criterion;
3. gather the smallest evidence that distinguishes them;
4. record the decision and accepted risk;
5. assign a follow-up owner and date if uncertainty remains.

The most senior person should not win by authority alone. A decision log is valuable when the team chooses a bounded approximation, accepts a migration window, or defers a cleanup with a real expiry.

## Review After Merge

Code review ends at merge, but the safety claim continues into rollout. Verify that the deployed artifact matches the reviewed revision, the migration ran as expected, metrics are visible, and stop conditions are active. Follow up on accepted risks and remove temporary branches when their evidence permits.

Post-incident review should ask whether the risk was visible before merge, whether evidence was missing, and whether the workflow made the safe path easy. The goal is system learning, not assigning blame to a reviewer who could not know an unobserved fact.

## Common Mistakes

* Reviewing style while ignoring authorization, data, effects, capacity, and rollback.
* Reading only the diff and not the contract, callers, migration, or deployment plan.
* Treating green unit tests or coverage as proof of integration correctness.
* Requesting a large rewrite when one focused invariant or evidence gap blocks safety.
* Giving every comment the same severity.
* Approving catch-all retries, silent fallbacks, or unbounded logs.
* Ignoring generated code, dependency updates, configuration, and operational ownership.
* Approving a schema change without mixed-version and recovery analysis.
* Treating a feature flag as a rollback plan without an owner and expiry.
* Using authority or review count to settle an empirical disagreement.
* Forgetting to verify the reviewed artifact and risk controls after merge.

## Senior Engineer Thinking

The senior question is not “does this diff look good?” It is “what contract and invariant changed, what can fail or be abused, what evidence supports the safety claim, who owns the durable and external effects, and how will we stop, roll back, or repair the change?”

Good review is focused skepticism with a path to confidence. It scales by matching depth to blast radius, separating blockers from preferences, asking for evidence at real boundaries, and leaving a durable decision when uncertainty cannot be eliminated.

## Exercises

1. Review a reservation insert diff and write one blocker, one question, and one non-blocking suggestion with consequences and evidence.
2. Build a review map for a database migration that changes schema, workers, caches, and an external message.
3. Convert five vague comments into actionable findings with severity, location, consequence, and verification.
4. Design an automated gate set and identify which safety questions still require a human reviewer.
5. Write a decision record for accepting a bounded stale-cache window and define owner, metric, stop condition, and expiry.

## Review Questions

* What makes code review a risk-assessment activity?
* Why should the contract and callers be read before individual lines?
* Which invariants require checking concurrency and failure paths?
* Why is authorization a query, cache, job, and repair concern?
* What evidence can unit tests provide, and what can they not prove?
* How should a reviewer distinguish a blocker from a preference?
* Which changes signal hidden ownership or operational scope?
* How can automation improve review without replacing judgment?
* What is a productive process for resolving disagreement?
* Why does review continue into rollout and post-merge observation?

## Summary

Code review is focused risk assessment against contracts, invariants, evidence, and recovery. Read context and callers, map inputs/data/effects/operations, check authorization and concurrency, inspect transactions and migrations, evaluate capacity and tests at the real boundary, make findings actionable and severity-aware, automate repeatable checks, resolve disagreement with evidence, and verify the reviewed artifact and controls after merge.

## References

- [Chapter 158 — Why Tests Exist](../../volumes/11-testing/158-why-tests-exist.md)
- [Chapter 170 — Test Design](../../volumes/11-testing/170-test-design.md)
- [Chapter 223 — Performance Mental Model](../../volumes/15-performance/223-performance-mental-model.md)
- [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../../volumes/16-distributed-systems/242-idempotency.md)
- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
