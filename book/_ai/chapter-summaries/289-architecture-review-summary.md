# AI Summary — Chapter 289 — Architecture Review

- Status: complete
- Volume: Volume 20 — SENIOR ENGINEERING
- Last updated: 2026-09-16

## Written material

The chapter distinguishes architecture review from code review and treats architecture as responsibility allocation across capabilities, dependencies, data, effects, deployment, and operations. It covers review packets and context maps, measurable quality attributes and fitness signals, dependency direction, ownership, consistency, external effects, security boundaries, PHP runtimes, capacity and failure domains, trade-offs, expand-and-contract migration, ADRs, a tenant-scoped search case study, workflow, exercises, and Chapter 290’s technical-debt handoff.

## Concepts already explained

Architecture review, responsibility allocation, context map, data owner, source of truth, derived state, quality attribute, fitness function, trust boundary, effect ownership, failure domain, mixed-version compatibility, expand-and-contract migration, forward recovery, accepted risk, and architecture decision record.

## Terminology established

Capability map, component/dependency map, data-ownership map, effect/timing map, deployment/operations map, fitness signal, ADR, and ownership table.

## Examples used

Tenant-scoped catalog search, reservation overlap, modular monolith versus service split, cache/search projection, payment-plus-notification workflow, PHP-FPM/CLI/worker capacity, and a rebuildable search projection ADR.

## Cross-references

The chapter links to [Chapter 240 — Backoff](../../volumes/16-distributed-systems/240-backoff.md), [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md), [Chapter 242 — Idempotency](../../volumes/16-distributed-systems/242-idempotency.md), [Chapter 251 — Eventual Consistency](../../volumes/16-distributed-systems/251-eventual-consistency.md), [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md), [Chapter 275 — Branch by Abstraction](../../volumes/18-legacy-php/275-branch-by-abstraction.md), [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md), [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md), and [Chapter 288 — Code Review](../../volumes/20-senior-engineering/288-code-review.md).

## Open threads

Continue with Chapter 290 — Technical Debt, turning accepted architectural compromises into explicit debt items with owners, consequences, and review conditions.

## Exact next section

Chapter 290 — Technical Debt: the Why This Matters section.

## Technical verification notes

The chapter contains illustrative text and one non-executable ADR template. Local Markdown links and required handoff files were checked, and `git diff --check` passed. The architecture guidance was proofread for ownership, data and effect boundaries, runtime/capacity concerns, migration, rollback, security, and Chapter 290’s handoff.

## Writing notes

Keep this summary short and update it after every writing session.
