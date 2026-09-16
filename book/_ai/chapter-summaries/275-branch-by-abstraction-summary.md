# AI Summary — Chapter 275 — Branch by Abstraction

- Status: complete
- Volume: Volume XVIII — LEGACY PHP
- Last updated: 2026-09-16

## Written material

Branch by Abstraction is presented as a stable capability interface that lets old and new implementations coexist while selection, authority, drift, and retirement remain explicit. The chapter covers interface design, safe installation sequence, typed read adapters and selectors, deterministic selection, side-effect-free shadow comparison, write authority, drift classification, transactions and effects, runtime and test boundaries, rollout and retirement, failure modes, exercises, and review questions.

## Concepts already explained

Branch by Abstraction, stable capability contract, legacy adapter, new adapter, selection policy, read authority, write authority, shadow comparison, drift classification, side-effect authority, operation identity, transaction ownership, branch lifecycle, and implementation retirement.

## Terminology established

Capability interface, compatibility adapter, selection context, policy version, safe default, shadow result, representation drift, data drift, policy drift, operational drift, environment drift, harness drift, authoritative writer, branch phase, retirement evidence, and recovery window.

## Examples used

The chapter includes abstraction-quality comparison, installation sequence, typed `InvoiceView`, `InvoiceReader`, legacy/new reader adapters, `ReadPath`, `ReaderSelector`, selection flow, shadow comparison, write-authority phase table, drift categories, transaction/effect guidance, branch lifecycle, failure modes, and abstraction-migration exercises.

## Cross-references

- [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../../volumes/16-distributed-systems/243-message-delivery.md)
- [Chapter 264 — Deployment](../../volumes/17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 270 — PHP 5 Codebases](../../volumes/18-legacy-php/270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](../../volumes/18-legacy-php/271-legacy-architecture.md)
- [Chapter 272 — Characterization Tests](../../volumes/18-legacy-php/272-characterization-tests.md)
- [Chapter 273 — Safe Refactoring](../../volumes/18-legacy-php/273-safe-refactoring.md)
- [Chapter 274 — Strangler Pattern](../../volumes/18-legacy-php/274-strangler-pattern.md)
- [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md)
- [Chapter 278 — PHP Version Migration](../../volumes/18-legacy-php/278-php-version-migration.md)

## Open threads

Continue Volume XVIII with Chapter 276 on Framework Migration, carrying forward capability contracts, adapters, selection authority, compatibility, and characterization evidence.

## Exact next section

Chapter 276 — Framework Migration: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the PHP example, local Markdown links resolved, and `git diff --check` passed. The chapter was proofread for interface scope, authority, shadow safety, drift classification, transaction/effect boundaries, runtime limitations, and retirement. No live application, database, queue, provider, traffic, or deployment integration was run.

## Writing notes

Chapter 274 covers capability routing and ownership transfer. Chapter 275 covers dual implementations behind a stable abstraction. Keep Laravel/Symfony lifecycle and adapter mechanics in Chapter 276, database transitions in Chapter 277, and runtime migration in Chapter 278.
