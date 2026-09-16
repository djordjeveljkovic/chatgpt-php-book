---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 290
title: Technical Debt
slug: technical-debt
status: complete
summary: ../../_ai/chapter-summaries/290-technical-debt-summary.md
---

# Chapter 290 — Technical Debt

## Why This Matters

Technical debt is a present shortcut, constraint, or deferred investment that increases the cost, risk, or uncertainty of future change. It is not a synonym for old code, ugly code, or code another engineer dislikes. A deliberate compatibility adapter can be rational debt during a migration; an elegant abstraction can become debt when it adds coupling without reducing risk.

Debt becomes dangerous when its interest is invisible. Delivery slows, incidents repeat, deployments become fragile, security exposure persists, and engineers spend more time reconstructing context than changing behavior. Chapter 289’s architecture review identifies trade-offs. This chapter turns the accepted consequences into items that can be measured, owned, prioritized, repaid, or deliberately contained.

## A Practical Mental Model

Use four questions for every proposed debt item:

1. What was deferred or constrained?
2. What recurring cost or risk does that create?
3. Who is affected, and how will we observe it?
4. What event would justify repayment, containment, or retirement?

Distinguish:

| Dimension | Examples |
| --- | --- |
| intention | explicit temporary shortcut versus accidental consequence |
| lifetime | migration bridge versus structural limitation |
| visibility | documented register item versus hidden workaround |
| surface | code, test, architecture, dependency, data, security, performance, or operations |

Debt is a management concept, not a moral judgment. The useful question is not “is this code clean?” but “what future option does this choice make more expensive, and is that cost worth carrying?”

## Principal, Interest, and Risk Premium

Principal is the work required to reduce or remove the debt. Interest is the recurring cost of keeping it: extra review time, manual repair, slow tests, deployment coordination, operational toil, or increased defect probability. A risk premium is the possible cost of a failure or compliance event whose probability is uncertain.

Useful measurements include:

* time spent fixing recurring incidents;
* lead time for changes in affected modules;
* rollback frequency and failed deployment rate;
* flaky-test reruns and time spent quarantining tests;
* deployment duration and manual release steps;
* dependency age, unsupported runtime versions, and vulnerability exposure;
* p95/p99 latency, query count and memory, connection use, and queue age;
* number of teams coupled to a schema or message;
* number of manual repair steps after a partial failure.

A rough decision aid can be written as:

~~~text
priority =
    frequency of impact
    × cost per occurrence
    × risk or uncertainty
    ÷ repayment effort
~~~

This is not a financial truth. It prevents a large, dramatic rewrite from automatically outranking a small issue that interrupts every deployment. Include strategic value, security exposure, deadlines, and reversibility when the simple model is insufficient.

## The Debt Register

Make debt durable enough to survive a sprint boundary:

~~~text
id:
title:
category:
affected capability:
description and origin:
current workaround:
principal estimate:
monthly interest:
failure modes:
security or compliance impact:
owner:
priority:
evidence:
repayment trigger:
review date:
retirement or containment condition:
~~~

“Clean this up” is not a debt record. The record should describe an observable consequence, a decision owner, and the evidence that would change its priority.

For example:

~~~text
id: DEBT-042
title: legacy order status strings
category: compatibility/data
principal: introduce an enum-compatible representation and migrate 14 consumers
interest: each new status requires a manual consumer search; the manual process was involved in two regressions last quarter
risk: an unrecognized value can be silently ignored by old PHP 7 workers
owner: Orders team
trigger: before adding another status or upgrading the worker fleet
review: 2026-12-01
~~~

The register is not a graveyard of complaints. Close an item when it is repaid, superseded, contained with evidence, or retired with the capability.

## Legacy PHP Compatibility Debt

Legacy runtime debt is a system constraint, not only a syntax problem. Inventory:

* dynamic properties, weak comparisons, implicit coercion, and legacy constructors;
* removed or deprecated functions and changed error behavior;
* extension availability and configuration differences;
* case-sensitive filesystem assumptions;
* old Composer constraints and autoload behavior;
* PHP-FPM, CLI, cron, and worker configuration drift;
* serialized values, timezone assumptions, and long-lived process state.

Use a bounded sequence:

1. characterize observed behavior and decide which behavior is actually required;
2. inventory runtime, extension, SAPI, and deployment assumptions;
3. add compatibility tests around invariants and negative paths;
4. introduce a narrow adapter or compatibility layer;
5. migrate callers incrementally and observe mixed versions;
6. remove the adapter only after consumer and recovery evidence is complete.

Rewriting every legacy class before understanding hidden contracts often converts visible debt into unknown risk. A temporary adapter is acceptable when its owner, removal condition, and review date are explicit.

## Migration and Data Debt

Data debt appears as shared tables without an owner, missing constraints, inconsistent identifiers, stale projections, unbounded backfills, unsafe dual writes, and schema changes that cannot coexist with old readers. Its interest is paid every time an engineer must guess which representation is authoritative.

Prefer expand and contract:

1. add a compatible column, constraint, message field, or interface;
2. deploy readers and writers that tolerate both versions;
3. backfill in bounded, resumable batches;
4. reconcile old and new representations and measure drift;
5. switch ownership with a stop condition;
6. remove compatibility code only after old consumers and repair paths are gone.

A resumable backfill needs a stable checkpoint, bounded work, and an explicit transaction scope. A production backfill also needs rate limits, retry classification, metrics, reconciliation, and a stop or pause control.

## Test Debt

Test debt has several different shapes:

* no tests around a business invariant;
* happy-path tests that omit invalid input, concurrency, or failure;
* tests coupled to implementation details and expensive to change;
* flaky tests that train engineers to rerun until green;
* slow suites that discourage local verification;
* missing characterization tests around legacy behavior;
* mocks that hide SQL, queue leases, provider ambiguity, or authorization scope.

Repay it by writing a test charter first. State the invariant, boundary, oracle, fixture controls, and failure cases. Add the smallest useful evidence: characterization for observed behavior, integration tests for database/cache/queue semantics, contract tests for messages, and operational tests for rollout or recovery. Increasing line coverage without improving the oracle is not repayment.

Chapters 272, 280, 284, and 287 provide examples: capture legacy behavior before refactoring, test reservation overlap under concurrency, test queue redelivery and idempotency, and exercise real SQL and keyset pagination at the database boundary.

## Architecture Debt

Architecture debt accumulates when responsibilities are duplicated or hidden:

* multiple modules enforce authorization differently;
* one module queries another module’s tables directly;
* a service locator hides dependency direction;
* provider calls run inside database transactions;
* queues lack idempotency, schema compatibility, or replay ownership;
* caches are treated as authoritative;
* no team owns a process, alert, migration, or repair action;
* a service boundary is introduced before a capability boundary exists.

Record the violated boundary and measurable consequence. “Catalog and search share too much” becomes useful when rewritten as “search writes product truth in two places, projection drift requires manual repair, and the next schema change must coordinate three deploys.” Repayment may be a modular boundary, an adapter, a single writer, or a runbook—not necessarily a new service.

## Dependency and Supply-Chain Debt

Dependency debt includes unsupported packages, broad Composer constraints, stale lock files, abandoned libraries, transitive vulnerabilities, incompatible extensions, framework coupling, and unreviewed upgrade jumps. “Latest” is not a remediation strategy; an upgrade can introduce new API, runtime, migration, and rollback work.

A safe repayment sequence is:

1. identify direct and transitive consumers and the supported PHP/platform range;
2. inspect changelogs, advisories, licenses, and compatibility claims;
3. update the manifest and lock file in a controlled change;
4. run characterization, integration, static, and deployment checks;
5. stage the artifact and observe mixed versions;
6. retain rollback or forward-recovery options for changed data and messages.

If a package cannot be replaced yet, contain it with an adapter, permission boundary, pinned version, vulnerability monitoring, and a trigger for reassessment.

## Security Debt

Security debt deserves priority based on exposure and consequence, not only how often engineers touch the code. Examples include plaintext or long-lived secrets, late tenant authorization, unsafe deserialization, missing audit trails, vulnerable dependencies, unbounded uploads or queries, and personal data in logs.

For each item, identify the trust boundary, affected principals, exploit path, detection signal, containment, and remediation. Restricting access, rotating credentials, adding a query limit, redacting logs, or isolating a component may be an immediate containment step while a larger migration is planned. A security debt item can outrank a high-frequency developer inconvenience because one event could expose every tenant.

## Performance Debt

PHP performance debt often hides in shared resources:

* loading entire files instead of streaming;
* N+1 queries or unindexed predicates;
* repeated expensive normalization;
* unbounded result sets;
* cache stampedes;
* memory growth in long-running workers;
* excessive PHP-FPM concurrency exhausting the database.

Measure a stated workload: p95/p99 latency, memory per worker, query count and rows examined, database connections, cache hit/fill behavior, queue age, and retry amplification. Average latency alone can hide a tail failure that saturates a shared dependency. Do not repay performance debt from intuition when a small workload experiment can distinguish a real bottleneck from an attractive theory.

## Operational Debt

Operational debt exists when normal traffic works but the team cannot safely diagnose or recover stress. Examples include missing readiness policy, weak metrics and logs, manual deployment steps, unclear alert ownership, unrecoverable queue failures, absent restore rehearsals, no graceful drain, and rollback plans that ignore schema or message compatibility.

Repayment can be a health contract, a bounded dashboard, a runbook, a replay tool with authorization, a restore drill, a deployment gate, or a forward-recovery procedure. The work is complete only when an operator can use it under pressure and the signal distinguishes failure from zero traffic or unknown collection.

## Repayment Strategies

Choose the smallest strategy that reduces the relevant interest:

| Strategy | Best fit | Main risk |
| --- | --- | --- |
| opportunistic repayment | related code is already changing | scope expands without evidence |
| dedicated maintenance | recurring interest is measurable | work is deferred repeatedly |
| characterization first | behavior is unknown | accidental behavior becomes permanent |
| branch by abstraction | callers can migrate incrementally | adapter and branch linger |
| strangler migration | one capability can move at a time | two systems drift during transition |
| incremental constraints | data must become safer gradually | old writers still bypass the rule |
| containment | repayment is not worth the principal now | hidden interest grows without review |
| retirement | capability has no future value | deletion misses consumers or data |

Every plan should name affected users, boundary, owner, evidence, rollback or forward recovery, stop condition, and removal date for temporary code. “We will clean it up later” is a prediction without a control.

## When Not to Repay Debt

Repayment may be irrational when the system is near retirement, the capability will soon be replaced, the interest is negligible, the code is isolated and stable, or remediation creates more migration risk than it removes. A large principal with little business value should not automatically consume a quarter.

Containment can be the responsible choice:

* freeze the interface and document the constraint;
* restrict access or isolate the component;
* add monitoring and a clear alert owner;
* cap resource use and disable risky paths;
* define a replacement trigger and review date.

Security, compliance, and reliability risks may still require repayment even when the feature is old. “Do not repay” means “make an explicit risk decision,” not “forget the item.”

## Common Misconceptions

| Misconception | Correction |
| --- | --- |
| all old code is debt | age is evidence only; measure current cost and risk |
| debt means bad code | debt includes data, architecture, dependencies, security, performance, and operations |
| more abstraction reduces debt | abstraction can add coupling, indirection, and maintenance cost |
| a rewrite is faster | rewrites discard hidden contracts unless behavior is characterized |
| coverage measures quality | coverage does not prove invariants, authorization, concurrency, or recovery |
| latest dependencies eliminate debt | upgrades create compatibility and migration work of their own |
| every item needs immediate repayment | prioritize impact, interest, risk, value, and reversibility |
| a TODO is a debt register | a record needs consequence, owner, evidence, trigger, and review |
| architecture debt is solved by microservices | network boundaries add latency, failure, deployment, and ownership costs |
| rollback always undoes migration | schema, messages, caches, and external effects may require forward recovery |

## Senior Engineer Thinking

Technical debt is a managed trade-off. Senior engineers make interest observable, distinguish risk from preference, choose repayment proportional to future value, and preserve the option to contain or retire a capability. They do not use debt language to shame a team or to justify a rewrite without a migration and recovery plan.

The decisive record is not “we accepted debt.” It is “we accepted this consequence, for this reason, with this owner, signal, trigger, and review date.” That record lets a later engineer decide with evidence rather than folklore.

## Exercises

1. Create a debt register for the Northstar legacy order portal from Chapter 279, including one compatibility, data, test, security, and operational item.
2. Estimate the monthly interest of a flaky importer suite using rerun time, delayed releases, and escaped failures.
3. Classify a shared database table as architecture and data debt, then name the first ownership boundary to establish.
4. Plan a PHP 7-to-8 compatibility migration with characterization, adapters, mixed-version evidence, and retirement criteria.
5. Prioritize three security and performance debt items using impact, frequency, risk, effort, and reversibility.
6. Design a bounded, resumable backfill with checkpoint, transaction, retry, reconciliation, and stop conditions.
7. Decide whether to repay or contain debt in a retired feature and write the evidence that would change the decision.
8. Convert three vague cleanup requests into debt records with measurable consequences and triggers.

## Review Questions

* What distinguishes technical debt from old or unattractive code?
* How do principal, interest, and risk premium guide prioritization?
* Which fields make a debt register actionable?
* Why can a compatibility adapter be rational debt?
* What makes data, test, dependency, security, performance, and operational debt different?
* Why do migrations need compatibility, checkpoints, reconciliation, and recovery?
* What evidence should be gathered before repaying performance debt?
* When is containment better than repayment?
* Why is a rewrite not automatically a debt-repayment strategy?
* What owner, trigger, and signal should every accepted debt item have?

## Summary

Technical debt is a present trade-off that raises future cost, risk, or uncertainty. Classify its surface and intention, measure principal and recurring interest, record consequences and ownership, prioritize security and operational exposure appropriately, use characterization and expand-and-contract migration for unknown behavior, repay with bounded evidence and recovery, and contain or retire debt deliberately when repayment is not valuable. A debt item is managed only when its owner, signal, trigger, and review date are visible.

## References

- [Chapter 270 — PHP 5 Codebases](../../volumes/18-legacy-php/270-php-5-codebases.md)
- [Chapter 272 — Characterization Tests](../../volumes/18-legacy-php/272-characterization-tests.md)
- [Chapter 273 — Safe Refactoring](../../volumes/18-legacy-php/273-safe-refactoring.md)
- [Chapter 275 — Branch by Abstraction](../../volumes/18-legacy-php/275-branch-by-abstraction.md)
- [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md)
- [Chapter 279 — Legacy Case Study](../../volumes/18-legacy-php/279-legacy-case-study.md)
- [Chapter 280 — Tennis Reservation Service](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md)
- [Chapter 284 — Queue Worker](../../volumes/19-small-engineering-projects/284-queue-worker.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 288 — Code Review](288-code-review.md)
- [Chapter 289 — Architecture Review](289-architecture-review.md)

## Chapter 291 Handoff

Technical debt often becomes visible first as a production symptom: a timeout, memory increase, queue backlog, noisy alert, failed deployment, or unexplained data discrepancy. Chapter 291 will show how to debug production safely by separating symptoms from causes, preserving evidence, narrowing hypotheses, and protecting the live system while investigating.

The next chapter should begin with:

> A production incident is not a puzzle to solve from intuition. It is a live system state that must be stabilized, measured, and reasoned about without destroying the evidence needed to find the cause.
