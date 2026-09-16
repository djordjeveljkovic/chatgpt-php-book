# AI Summary — Chapter 241 — Partial Failure

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Explains partial failure as expected distributed behavior. Covers explicit state machines, failure categories, ownership and evidence, partial results, multi-step workflows, compensation and reconciliation, PHP observations, testing, security, and operational mistakes.

## Concepts already explained

Partial failure, unknown completion, required and optional branch, operation evidence, compensation, reconciliation, stale event, and fail-closed recovery.

## Terminology established

Operation state, durable evidence, workflow transition, recovery action, partial response, and business invariant owner.

## Examples used

Operation and fan-out state diagrams, a typed OperationResult, checkout workflow, partial-result policy, and failure-injection tests.

## Cross-references

Chapters 238 and 234, plus the next chapter on idempotency.

## Open threads

Continue with operation identity and duplicate convergence in Chapter 242.

## Exact next section

Chapter 242 — Idempotency: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP example. Local Markdown links resolved and git diff --check passed. Live distributed integration was not run.

## Writing notes

Treats unknown completion as a distinct state and makes recovery evidence part of the design.
