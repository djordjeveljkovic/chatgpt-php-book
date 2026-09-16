# AI Summary — Chapter 271 — Legacy Architecture

- Status: complete
- Volume: Volume XVIII — LEGACY PHP
- Last updated: 2026-09-16

## Written material

Legacy architecture is presented as the effective graph of responsibilities, data, lifecycles, effects, and ownership rather than a folder tree or intended diagram. The chapter compares intended, static, and observed architecture; describes shared bootstraps, page-centered code, shared database modules, and scheduled workflows; identifies hidden coupling; defines responsibility, data, process, effect, and ownership boundaries; catalogs dependencies; explains dependency direction, transaction ownership, runtime/process differences, bounded observability, and incremental seams; and covers dual-path risks, failure modes, exercises, and review questions.

## Concepts already explained

Effective architecture, intended/static/observed views, call/data/lifecycle/effect edge, shared bootstrap, page-centered workflow, database as module boundary, scheduled workflow, hidden coupling, responsibility boundary, data owner, process boundary, effect boundary, ownership boundary, dependency direction, adapter seam, transaction ownership, shadow read, dual write, and architectural evidence.

## Terminology established

Architecture graph, bounded capability, dependency map, observed dependency, unknown edge, invariant owner, process owner, effect identity, recovery evidence, application operation, domain decision, repository port, effect port, infrastructure adapter, coexistence period, authority, reconciliation, removal condition, and architecture catalog.

## Examples used

The chapter includes a legacy dependency graph, intended/static/observed comparison table, common legacy architecture shapes, hidden-coupling search surfaces, a dependency-evidence flow, responsibility/data/process/effect/ownership boundaries, a typed `ArchitectureEdge` enum and readonly `ArchitectureDependency` class, accidental and inward dependency-direction diagrams, transaction and ownership analysis, web/CLI/cron/queue comparison, observability fields, an incremental change loop, migration-seam guidance, failure modes, and architectural exercises.

## Cross-references

- [Chapter 157 — Supply-Chain Security](../../volumes/10-security/157-supply-chain-security.md)
- [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../../volumes/16-distributed-systems/243-message-delivery.md)
- [Chapter 249 — Distributed Locks](../../volumes/16-distributed-systems/249-distributed-locks.md)
- [Chapter 256 — PHP-FPM](../../volumes/17-production-engineering/256-php-fpm.md)
- [Chapter 258 — Configuration](../../volumes/17-production-engineering/258-configuration.md)
- [Chapter 260 — Logging](../../volumes/17-production-engineering/260-logging.md)
- [Chapter 261 — Metrics](../../volumes/17-production-engineering/261-metrics.md)
- [Chapter 262 — Tracing](../../volumes/17-production-engineering/262-tracing.md)
- [Chapter 264 — Deployment](../../volumes/17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 270 — PHP 5 Codebases](../../volumes/18-legacy-php/270-php-5-codebases.md)
- [Chapter 272 — Characterization Tests](../../volumes/18-legacy-php/272-characterization-tests.md)
- [Chapter 274 — Strangler Pattern](../../volumes/18-legacy-php/274-strangler-pattern.md)
- [Chapter 275 — Branch by Abstraction](../../volumes/18-legacy-php/275-branch-by-abstraction.md)

## Open threads

Continue Volume XVIII with Chapter 272 on characterization tests, carrying forward the architecture maps, observed dependency evidence, ownership boundaries, and safe seams established here.

## Exact next section

Chapter 272 — Characterization Tests: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the PHP example, local Markdown links resolved, and `git diff --check` passed. The chapter uses architecture reasoning and existing local references; no live legacy application, database, queue, provider, deployment, or migration integration was run.

## Writing notes

Chapter 270 established what to inventory in a PHP 5 system. Chapter 271 turns that inventory into observed architecture maps, ownership decisions, explicit dependency direction, and incremental seams. Keep characterization-test mechanics in Chapter 272 and detailed refactoring or migration patterns in Chapters 273–278.
