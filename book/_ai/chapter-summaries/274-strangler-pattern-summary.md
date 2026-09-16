# AI Summary — Chapter 274 — Strangler Pattern

- Status: complete
- Volume: Volume XVIII — LEGACY PHP
- Last updated: 2026-09-16

## Written material

The Strangler Pattern is presented as staged capability and ownership migration, not merely a proxy in front of a rewrite. The chapter covers slice selection and charters, boundary preservation, read-only shadows, one authoritative writer, shared data ownership, deterministic route selection, staged rollout, rollback as a data decision, observability, retirement, failure modes, exercises, and review questions.

## Concepts already explained

Strangler Pattern, capability slice, slice charter, migration boundary, route authority, read shadow, write authority, coexistence, data ownership, compatibility translator, deterministic route policy, rollout stage, rollback compatibility, forward recovery, retirement, and migration evidence.

## Terminology established

Slice, entry-point contract, shadow path, authoritative writer, compatibility phase, route decision, policy version, failure domain, in-flight work, effect authority, reconciliation, retirement condition, old-path drain, and capability ownership transition.

## Examples used

The chapter includes a strangler boundary diagram, slice charter, read-migration flow, write-authority phase table, typed `RouteTarget`, `RouteDecision`, and `chooseTarget()` policy, staged rollout sequence, rollback questions, observability fields, retirement checklist, failure modes, and migration exercises.

## Cross-references

- [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../../volumes/16-distributed-systems/243-message-delivery.md)
- [Chapter 249 — Distributed Locks](../../volumes/16-distributed-systems/249-distributed-locks.md)
- [Chapter 264 — Deployment](../../volumes/17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 269 — Incident Response](../../volumes/17-production-engineering/269-incident-response.md)
- [Chapter 270 — PHP 5 Codebases](../../volumes/18-legacy-php/270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](../../volumes/18-legacy-php/271-legacy-architecture.md)
- [Chapter 272 — Characterization Tests](../../volumes/18-legacy-php/272-characterization-tests.md)
- [Chapter 273 — Safe Refactoring](../../volumes/18-legacy-php/273-safe-refactoring.md)
- [Chapter 275 — Branch by Abstraction](../../volumes/18-legacy-php/275-branch-by-abstraction.md)
- [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md)

## Open threads

Continue Volume XVIII with Chapter 275 on Branch by Abstraction, carrying forward explicit contracts, characterization evidence, narrow seams, route authority, and rollback boundaries.

## Exact next section

Chapter 275 — Branch by Abstraction: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the PHP example, local Markdown links resolved, and `git diff --check` passed. The chapter was proofread for capability slicing, route authority, shared-state compatibility, effect safety, deterministic policy, staged rollout, rollback, and retirement. No live traffic, database, queue, provider, deployment, or migration integration was run.

## Writing notes

Chapter 274 covers capability-by-capability migration and authority transitions. Keep dual implementations, feature selection, and drift-control mechanics in Chapter 275; framework, database, and PHP-version migration details remain in Chapters 276–278.
