# AI Summary — Chapter 290 — Technical Debt

- Status: complete
- Volume: Volume 20 — SENIOR ENGINEERING
- Last updated: 2026-09-16

## Written material

The chapter defines technical debt as a present trade-off that increases future cost, risk, or uncertainty. It distinguishes intentional, accidental, temporary, structural, visible, and hidden debt; explains principal, interest, risk premium, debt registers, evidence, prioritization, and repayment; covers PHP compatibility, data/schema, test, architecture, dependency, security, performance, and operational debt; compares repayment and containment; and hands off to Chapter 291 on production debugging.

## Concepts already explained

Technical debt, principal, interest, risk premium, debt register, debt item, repayment trigger, review condition, containment, retirement, compatibility debt, data debt, test debt, architecture debt, dependency debt, security debt, performance debt, operational debt, and expand-and-contract repayment.

## Terminology established

Debt priority aid, debt-record template, legacy order-status item, compatibility migration sequence, backfill requirements, evidence-based prioritization, repayment-strategy table, and containment decision.

## Examples used

Northstar legacy order portal, tenant-scoped catalog/search application, PHP-FPM/CLI/worker compatibility, shared product tables, search projection, cache, queue worker, notification provider, flaky importer tests, and bounded backfill planning.

## Cross-references

The chapter links to [Chapter 270 — PHP 5 Codebases](../../volumes/18-legacy-php/270-php-5-codebases.md), [Chapter 272 — Characterization Tests](../../volumes/18-legacy-php/272-characterization-tests.md), [Chapter 273 — Safe Refactoring](../../volumes/18-legacy-php/273-safe-refactoring.md), [Chapter 275 — Branch by Abstraction](../../volumes/18-legacy-php/275-branch-by-abstraction.md), [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md), [Chapter 279 — Legacy Case Study](../../volumes/18-legacy-php/279-legacy-case-study.md), [Chapter 280 — Tennis Reservation Service](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md), [Chapter 284 — Queue Worker](../../volumes/19-small-engineering-projects/284-queue-worker.md), [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md), [Chapter 288 — Code Review](../../volumes/20-senior-engineering/288-code-review.md), and [Chapter 289 — Architecture Review](../../volumes/20-senior-engineering/289-architecture-review.md).

## Open threads

Continue with Chapter 291 — Debugging Production, beginning with stabilization, measurement, evidence preservation, hypothesis narrowing, and safe live-system investigation.

## Exact next section

Chapter 291 — Debugging Production: the Why This Matters section.

## Technical verification notes

The chapter contains illustrative text and structured debt-register templates. Local Markdown links and required handoff files were checked, and `git diff --check` passed. The chapter was independently proofread for debt classification, principal/interest reasoning, PHP-specific migration concerns, security and operational priority, repayment versus containment, and the Chapter 291 handoff.

## Writing notes

Keep this summary short and update it after every writing session.
