---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 297
title: Senior PHP Interview Questions
slug: senior-php-interview-questions
status: complete
summary: ../../_ai/chapter-summaries/297-senior-php-interview-questions-summary.md
---

# Chapter 297 — Senior PHP Interview Questions

## Why This Matters

A senior interview answer explains the invariant, states assumptions and trade-offs, identifies failure modes, describes observability and recovery, and adapts the design to PHP’s runtime and operational boundaries. It is not a performance of memorized syntax trivia.

The strongest answer may begin with a question: which users, workload, data, compatibility window, and failure behavior are in scope? Clarifying the problem is part of solving it.

## A Repeatable Answer Structure

Use this sequence for coding, architecture, debugging, security, and system-design prompts:

1. clarify requirements and user impact;
2. state invariants and hard constraints;
3. propose the simplest safe design;
4. compare alternatives and inaction;
5. analyze failure, concurrency, and recovery;
6. describe testing and observability;
7. explain migration and rollout;
8. identify evidence that would change the answer.

This structure demonstrates judgment without requiring the candidate to guess the interviewer’s preferred architecture.

## PHP and Runtime Questions

Ask what changes between PHP-FPM request workers, CLI commands, cron, and long-running queue workers. A strong answer distinguishes request lifetime from process lifetime, `memory_limit` from RSS and container memory, web timeout from database/provider completion, OPcache and artifact identity, extension/SAPI configuration, Composer constraints and lockfiles, and mixed-version deployment.

Useful prompts:

* Why can increasing `pm.max_children` make the database slower?
* What state can leak between queue messages in a long-running worker?
* What does an OPcache or PHP-version mismatch look like during rollout?
* Which compatibility checks differ between FPM and CLI?
* What does a timeout tell you about an external effect?

The answer should explain consequences and evidence, not merely define the terms.

## Correctness and Invariants

Ask what must never break. Expected invariants include tenant isolation, object-level authorization, transaction and state-transition rules, idempotency for retries and external effects, stable ordering and pagination, data integrity, bounded resource use, and required auditability.

An elegant answer that violates an invariant should score poorly regardless of its latency or simplicity. Require the candidate to say where each invariant is enforced: query, constraint, transaction, cache key, job, provider boundary, or repair path.

## Architecture and Trade-Offs

Prompts should compare a database query, cache, projection, or service; synchronous notification with an outbox and queue; modular monolith with service extraction; strong with eventual consistency; build with buy; refactor with rewrite; and more FPM workers with database capacity.

Listen for latency, throughput, consistency, security, operational ownership, migration cost, reversibility, exit cost, and cost of inaction. “Use microservices” or “add a cache” is not an explanation.

## Failure-Mode Reasoning

Ask what happens when the database is unavailable or locked, Redis is stale or unavailable, a provider times out after completing, a queue delivers twice, an old worker reads a new message, a deployment partially succeeds, a projection needs rebuilding, a migration cannot roll back, or authorization changes while work is queued.

Strong candidates distinguish not started, failed, completed, completed externally, and completion unknown. They identify idempotency identity, durable evidence, reconciliation, dead letters, rollback limits, and forward recovery.

## Observability and Verification

“I would monitor it” is insufficient. Ask for the signal, baseline, alert condition, owner, and response. Useful evidence includes latency distributions, error and saturation metrics, structured logs and correlation IDs, traces across FPM/SQL/cache/queue/provider, query plans and lock data, queue age and retries, worker memory and restarts, authorization tests, tenant-isolation tests, and deployment/migration/rollback signals.

The candidate should state telemetry limits: sampling, cardinality, missing exporters, privacy, clock skew, and the difference between no failure and unavailable evidence.

## Security and Tenant Isolation

Use questions where a request supplies an object ID, tenant ID, callback URL, upload, webhook, or export request. Strong answers authorize the action against the resource, scope queries before loading data, preserve tenant scope in caches/jobs/indexes/exports/repair scripts, validate URLs against redirects and private destinations, protect secrets and logs, and include abuse limits and replay protection.

Authentication establishes identity; authorization decides whether that identity may act. Hidden UI controls and client-provided tenant IDs are not access control.

## Migration and Operations

Require discussion of expand-and-contract schema changes, old/new application compatibility, Composer and PHP-version migration, resumable backfills, queue-message compatibility, feature flags, canaries, rollback limitations, forward recovery, reconciliation, ownership, alerts, and runbooks. A candidate need not provide every command, but should notice when a choice is difficult to reverse.

## How to Assess an Answer

| Dimension | Strong answer | Weak answer |
| --- | --- | --- |
| requirements | clarifies scope and user impact | assumes an idealized problem |
| correctness | names and protects invariants | optimizes the happy path |
| trade-offs | compares options and inaction | presents one fashionable solution |
| failure | handles partial failure and unknown outcomes | says retry or rollback reflexively |
| security | scopes authorization across paths | trusts IDs or hidden controls |
| operations | names metrics, rollout, and recovery | ends at “the code works” |
| PHP expertise | distinguishes FPM, CLI, workers, Composer, extensions | recites trivia |
| evidence | proposes tests or measurements | relies on confidence |
| communication | states assumptions and uncertainty | overstates certainty |

Follow up with: “What invariant does this protect?”, “What fails first under load?”, “How would you detect that?”, “What if the provider completed after timeout?”, “How do old workers coexist?”, “Where is tenant scope enforced?”, “What would change your design?”, and “How do you recover if rollback is unsafe?”

## Case Study: Search Performance

Ask the candidate to improve Chapter 287’s tenant-scoped search. p99 rises for large tenants; FPM workers become occupied; SQL plans degrade; cache misses repeat queries; and queue-backed indexing falls behind.

The candidate should compare an indexed query, cache, projection, and service split while preserving tenant isolation, authorization, stable cursor ordering, bounded work, freshness, rebuildability, queue compatibility, FPM capacity, and database limits. Removing a tenant predicate or adding workers without checking connections is a deliberately tempting but unsafe answer.

## Common Mistakes

* treating the chapter as a list of PHP syntax questions;
* answering before clarifying workload and constraints;
* confusing consistency with correctness;
* optimizing averages instead of tail behavior;
* ignoring deployment, migration, and rollback;
* trusting a benchmark as production evidence;
* saying retry or rollback without unknown-outcome analysis;
* sacrificing authorization, isolation, idempotency, or bounded work;
* choosing architecture by fashion;
* scoring confidence instead of reasoning and evidence.

## Senior Engineer Thinking

Senior expertise is transferable reasoning. The candidate explains what must remain true, what can fail, how to observe it, which trade-off is being accepted, who owns recovery, and what evidence would change the decision. Precise uncertainty is stronger than invented certainty.

## Exercises

1. Answer a PHP-FPM versus CLI lifecycle question with resource and deployment consequences.
2. Design a reservation operation with authorization, overlap, idempotency, and unknown completion.
3. Compare query, cache, projection, and service options for tenant-scoped search.
4. Explain how to migrate a legacy application and Composer graph across PHP versions.
5. Review an upload, webhook, export, and repair path for security boundaries.
6. Build a monitoring and recovery answer for a queue backlog.
7. Create a rubric for assessing an architecture answer without prescribing one pattern.
8. Rewrite an overconfident response with assumptions, evidence, and a review condition.

## Review Questions

* What makes an answer senior rather than encyclopedic?
* Which invariants should a candidate name first?
* How do PHP-FPM and long-running workers change the answer?
* Why should inaction and containment be options?
* What evidence distinguishes a failure from unknown completion?
* Which signals establish operational readiness?
* How does tenant isolation extend beyond the database query?
* When is rollback unsafe?
* What makes a decision reversible?
* How should an interviewer score reasoning rather than preference?

## Summary

Senior PHP interview answers clarify requirements, state invariants, choose proportionate complexity, compare options and inaction, reason through failure and recovery, preserve security and tenant isolation, adapt to PHP runtime boundaries, name evidence and observability, plan migration and rollout, and communicate uncertainty. The goal is explainable engineering judgment, not trivia.

## References

- [Chapter 223 — Performance Mental Model](../../volumes/15-performance/223-performance-mental-model.md)
- [Chapter 270 — PHP 5 Codebases](../../volumes/18-legacy-php/270-php-5-codebases.md)
- [Chapter 280 — Tennis Reservation Service](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 288 — Code Review](288-code-review.md)
- [Chapter 292 — Performance Investigation](292-performance-investigation.md)
- [Chapter 293 — Security Review](293-security-review.md)
- [Chapter 294 — Incident Investigation](294-incident-investigation.md)

## Chapter 298 Handoff

Interview answers become more precise when candidates can state which behavior depends on the PHP version, extensions, runtime, or deployment environment. Chapter 298 begins the reference volume with a PHP Version Matrix.
