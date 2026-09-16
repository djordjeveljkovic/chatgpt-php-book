---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 273
title: Safe Refactoring
slug: safe-refactoring
status: complete
summary: ../../_ai/chapter-summaries/273-safe-refactoring-summary.md
---

# Chapter 273 — Safe Refactoring

## Why This Matters

Refactoring changes a program’s internal structure while preserving a chosen external behavior. In a legacy PHP system, that preservation claim is difficult because the boundary is wide: include order, globals, SQL side effects, output, warnings, files, cron timing, and provider calls may all be observable.

The safe unit is not a large cleanup branch. It is a small, reviewable transformation with a known baseline, a bounded risk, and evidence that the intermediate state still works. The purpose is to reduce coupling one step at a time so that later changes become cheaper and more visible.

Refactoring does not mean refusing to fix defects. It means separating a behavior-preserving transformation from an intentional behavior change. If a security or data defect must change now, record that as a deliberate contract change and test the new requirement rather than hiding it inside a cleanup.

## Mental Model

Use a loop with a safe stopping point:

~~~text
baseline
   ↓
one small transformation
   ↓
run checks and characterization tests
   ↓
inspect behavior and diff
   ↓
commit a reversible step or revert
~~~

Every step should answer:

* What behavior is being preserved?
* Which dependency or risk is being reduced?
* What is the smallest code change that tests the hypothesis?
* Which checks can detect drift?
* What is the recovery action if the result is wrong?

If the team cannot answer these questions, the change is discovery or redesign work, not yet a safe refactoring.

## Refactoring, Repair, and Rewrite

Name the kind of change before editing:

| Change | Primary claim | Evidence needed |
| --- | --- | --- |
| Refactoring | Observable behavior remains within the named contract. | characterization tests and focused checks |
| Repair | A defect is intentionally corrected. | requirement, regression test, impact review |
| Redesign | Responsibility, ownership, or boundary changes. | architecture decision and transition plan |
| Rewrite | A replacement is built with a new implementation and compatibility plan. | differential evidence, rollout, recovery |

One pull request can contain more than one kind, but separate commits and explanations make the contract change visible. A renamed method mixed with a payment-rule change is harder to review than two explicit steps.

## Establish the Baseline

Before changing code, capture the behavior that matters:

1. choose one entry point and capability;
2. record inputs, runtime, configuration, and fixture state;
3. capture return values, output, errors, writes, messages, and provider effects;
4. classify each observation as required, accidental, unsafe, or unknown;
5. run the baseline more than once when nondeterminism is possible;
6. save the command, environment, test result, and known limitations.

Chapter 272 covers characterization in detail. A baseline is not “the whole application passes.” It is a named observation under stated conditions. Do not broaden the change merely because an unrelated test is flaky.

## Select a Low-Risk Transformation

Good first transformations reduce one source of ambiguity:

* rename a local variable while preserving all calls and output;
* extract a pure calculation from a page script;
* replace a duplicated literal with one named constant;
* introduce a parameter for a hidden global read;
* wrap one legacy function in an adapter without changing its result;
* make a dependency injectable at one entry point;
* split a query-building step from result interpretation;
* add an explicit return value when existing callers ignore the return value and the observed return contract is unchanged.

Avoid making the first step a framework upgrade, schema rewrite, namespace conversion, and business-rule change. Those may be necessary later, but they create too many explanations for one result.

## Write a Change Charter

Before the baseline, write a short charter for the proposed step:

~~~text
capability: send invoice receipt
preserved behavior: missing orders return "not-found"; one receipt is attempted
dependency reduced: page code no longer reads the database global directly
risks: mail ordering, transaction timing, PHP 5 entry point
evidence: characterization cases, adapter test, CLI/web smoke checks
stop condition: any changed effect count or authorization result
rollback: restore the previous caller while retaining recorded observations
~~~

The charter names the reason for the change and its stopping conditions. It is not a promise that the legacy behavior is desirable. If the step discovers that the stated contract is false or incomplete, stop and update the evidence before continuing.

## Break Dependencies at the Narrowest Seam

Start where the legacy detail enters the capability:

~~~text
legacy page
   ↓
small boundary adapter
   ↓
explicit operation
   ↓
existing database/file/provider behavior
~~~

The adapter should own translation, not business policy. It can read a global and pass a value to a modern operation, translate an old result shape, or record an external effect. The old implementation remains behind the seam until tests and ownership justify moving it.

A seam is useful when it reduces the number of callers that know a legacy detail. If every caller still reaches the global through a service locator, the dependency is only hidden. If the adapter changes error, encoding, transaction, or authorization semantics, it is a migration step and needs a stronger contract.

## A Behavior-Preserving Seam

This modern operation depends on explicit ports while a legacy adapter can preserve the old database and audit behavior:

~~~php
<?php

declare(strict_types=1);

interface OrderStore
{
    public function reserve(string $orderId): bool;
}

interface EventRecorder
{
    /** @param array<string, scalar> $context */
    public function record(string $name, array $context): void;
}

final readonly class ReserveOrder
{
    public function __construct(
        private OrderStore $orders,
        private EventRecorder $events,
    ) {
    }

    public function __invoke(string $orderId): bool
    {
        $reserved = $this->orders->reserve($orderId);

        $this->events->record('order.reserve', [
            'order_id' => $orderId,
            'outcome' => $reserved ? 'reserved' : 'rejected',
        ]);

        return $reserved;
    }
}
~~~

The operation does not know whether the old system uses a global connection, a legacy database function, PDO, a stored procedure, or a test fake. That is useful only if the adapter preserves the contract: the same authorization decision, transaction timing, error classification, and effect count. The interfaces are a seam, not proof that the old behavior has been improved.

## Globals, Static State, and Include Order

Make hidden inputs explicit one at a time. A safe sequence is:

1. add a parameter or port beside the global read;
2. pass the current global value through the new seam;
3. characterize both paths;
4. switch one caller;
5. remove the global read only after all callers are accounted for.

Do not replace a global with a mutable singleton and call the work complete. A singleton can preserve the same lifecycle, tenant leakage, initialization order, and test isolation problems under a more respectable name.

When includes define functions, constants, or variables, first document the bootstrap contract. Then make the smallest order-preserving extraction. Tests should exercise the real entry point as well as the extracted unit; otherwise an include-order regression can remain invisible.

## Database and Transaction Boundaries

Moving SQL is not a behavior-preserving refactor if it changes:

* connection charset, timezone, or SQL mode;
* transaction start, commit, rollback, or autocommit behavior;
* locking order, affected-row interpretation, or retry behavior;
* result ordering, missing-versus-null values, or duplicate rows;
* trigger, audit, outbox, or cache side effects.

Before extracting a repository, document who owns the transaction and the invariant. Keep the first change inside the existing transaction if possible. If ownership must move, treat the change as a transition with characterization, reconciliation, and recovery evidence. Chapter 277 covers database migration mechanics.

## External Effects and Ordering

Refactoring a provider call or mail send requires more than matching its arguments. Preserve or deliberately change:

* whether the effect is attempted before or after commit;
* operation identity and duplicate handling;
* timeout and unknown-completion behavior;
* retry and reconciliation policy;
* payload shape and sensitive-field handling;
* ordering relative to files, messages, and audit records.

Use recording adapters in tests. Do not invoke a real provider to prove that a refactor is safe. A method that returns successfully can still have caused a duplicate external effect.

## Static Analysis and Runtime Checks

Static analysis helps reveal unused symbols, wrong types, unreachable code, and dependency direction, but it cannot observe dynamic includes, database triggers, runtime configuration, or provider completion. A clean analyzer result is evidence about the analyzed model, not a full compatibility guarantee.

Run checks at the relevant layers:

* syntax and static analysis for the target source/runtime;
* characterization tests at the real entry point;
* focused unit tests for extracted decisions;
* database and filesystem tests for preserved boundaries;
* queue, CLI, cron, and worker tests for process behavior;
* deployment and rollback checks when the artifact or startup changes.

Modern PHP syntax in an adapter or test harness cannot be loaded by PHP 5. Keep the boundary explicit and test the old application with its exact runtime when compatibility is part of the claim.

## Performance and Capacity

A refactor can preserve output while changing resource behavior. Compare query count, rows scanned, memory lifetime, lock duration, request time, worker retention, and provider calls when the capability is sensitive to them.

Do not accept a microbenchmark that omits the database, filesystem, serialization, or process lifetime that dominates production. Conversely, do not reject a structural improvement because a local benchmark includes unrelated startup noise. Chapter 223 covers measurement and Chapter 235 covers capacity reasoning.

## Reversible Commits and Review

A reversible refactoring commit has a narrow purpose, a stable build state, and a clear rollback action. Prefer:

~~~text
commit 1: add characterization and seam
commit 2: switch one caller
commit 3: remove duplicate path
commit 4: intentional contract change, if needed
~~~

Keep generated files and formatting-only changes separate when they obscure the behavior change. Review the diff for accidental SQL, route, permission, configuration, and public-signature changes. A revert is useful only if the intermediate state is deployable and data or external effects are compatible with the rollback.

## A Safe Refactoring Checklist

Before merging, answer:

* What named behavior is preserved?
* Which characterization or regression tests cover it?
* What hidden input or side effect was discovered?
* Which boundary became more explicit?
* Who owns the data, transaction, effect, and rollback?
* What exact runtime and process types were checked?
* What can be observed after deployment?
* What happens if old and new code coexist?
* Which state cannot be reverted automatically?
* When can the temporary seam be removed?

If these answers are unknown, record the unknown and reduce the scope until the next step is safe enough to learn from.

## Common Mistakes

* Calling a rewrite a refactor because both versions return similar HTML.
* Combining formatting, framework upgrades, schema changes, and rule changes in one diff.
* Extracting a class without capturing behavior first.
* Moving a query without checking transaction, ordering, charset, or trigger behavior.
* Replacing a global with a service locator or singleton.
* Mocking every boundary and never checking the real adapter contract.
* Treating static analysis as proof of runtime compatibility.
* Ignoring CLI, cron, queue, worker, and admin entry points.
* Measuring only latency while increasing database or provider load.
* Reverting code after irreversible external effects without reconciliation.
* Keeping temporary adapters with no owner or removal condition.

## Senior Engineer Thinking

The senior question is not “how much old code can be cleaned up in this sprint?” It is “what is the smallest transformation that reduces one risk while leaving the system understandable and recoverable?”

Safe refactoring is disciplined uncertainty reduction. Establish an observation, change one relationship, inspect the result, and leave a durable seam or remove it when its purpose is complete. Respect existing contracts without treating every accident as sacred; make intentional changes explicit, especially when security, data, or external effects are involved.

## Exercises

1. Select one Chapter 272 characterization case and propose three refactoring steps, each with a baseline, risk, check, and rollback action.
2. Find a global database handle. Design a seam that makes the connection explicit while preserving transaction and error behavior.
3. Review a proposed SQL extraction. List every behavior besides returned rows that must be characterized.
4. Split a mixed commit into behavior-preserving refactoring, operational change, and intentional business-rule repair.
5. Design a commit sequence for replacing one mail adapter. Include operation identity, unknown completion, recording tests, deployment coexistence, and reconciliation.

## Review Questions

* What makes a refactoring step safe enough to review and revert?
* How does a refactor differ from a repair, redesign, or rewrite?
* Why is a baseline narrower than “the application passes”?
* What makes a seam reduce coupling rather than hide it?
* Which database behaviors can change when SQL is moved?
* Why do external effects require identity and recovery evidence?
* What can static analysis fail to observe in legacy PHP?
* How can a refactor preserve output but damage capacity?
* Why should reversible commits leave a deployable intermediate state?
* When should a behavior change be separated from a refactoring commit?

## Summary

Safe refactoring is a small, observable, behavior-preserving transformation with a named baseline, bounded risk, and recovery action. Distinguish refactoring from repair and redesign, characterize the real boundary, break hidden dependencies at narrow seams, preserve database and external-effect contracts, test exact runtime and process behavior, measure resource consequences, and keep commits reversible. The goal is not maximum cleanup; it is reducing one architectural risk while making the next change safer and more verifiable.

## References

- [Chapter 165 — Test Doubles](../11-testing/165-test-doubles.md)
- [Chapter 170 — Test Design](../11-testing/170-test-design.md)
- [Chapter 223 — Performance Mental Model](../15-performance/223-performance-mental-model.md)
- [Chapter 235 — Scaling](../15-performance/235-scaling.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 260 — Logging](../17-production-engineering/260-logging.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](./271-legacy-architecture.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 274 — Strangler Pattern](./274-strangler-pattern.md)
- [Chapter 275 — Branch by Abstraction](./275-branch-by-abstraction.md)
