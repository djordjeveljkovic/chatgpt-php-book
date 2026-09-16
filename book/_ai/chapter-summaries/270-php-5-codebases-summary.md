# AI Summary — Chapter 270 — PHP 5 Codebases

- Status: complete
- Volume: Volume XVIII — LEGACY PHP
- Last updated: 2026-09-16

## Written material

PHP 5 codebases are treated as legacy systems spanning source syntax, the exact PHP runtime, extensions, dependencies, configuration, SAPI, process lifecycle, deployment, data boundaries, and operational contracts. The chapter covers version and runtime inventory, hidden contracts, compatibility boundaries, evidence catalogs, narrow compatibility layers, security containment, database and charset/transaction behavior, PHP-FPM/CLI/cron/queue differences, testing under exact runtimes, performance, concurrency, migration seams, and the separation of runtime, framework, database, and business changes.

## Concepts already explained

PHP 5 codebase, exact compatibility target, runtime inventory, SAPI, extension boundary, hidden contract, compatibility boundary, legacy API, compatibility layer, evidence catalog, immediate containment, unsupported runtime, data-boundary risk, operational assumption, migration seam, characterization evidence, and explicit transition.

## Terminology established

Runtime boundary, source boundary, environment boundary, operations boundary, version drift, extension inventory, configuration inventory, dependency inventory, legacy worker, adapter seam, owner and expiry, unknown evidence, SAPI mismatch, process lifetime, charset contract, transaction contract, side-effect boundary, compatibility risk, containment control, and rollback-safe transition.

## Examples used

The chapter includes a four-boundary legacy-system model, a runtime inventory table, PHP-version compatibility notes, search surfaces for legacy APIs and implicit behavior, typed `LegacyRisk` and `LegacyFinding` records, an immediate-containment policy, compatibility-layer guidance, security and data-boundary checklists, PHP-FPM/CLI/cron/queue testing distinctions, PHP 5 compatibility testing, performance and concurrency reasoning, migration boundaries, failure modes, senior-engineer questions, and review exercises.

## Cross-references

- [Chapter 92 — Composer](../../volumes/07-composer-and-the-php-ecosystem/092-composer.md)
- [Chapter 157 — Supply-Chain Security](../../volumes/10-security/157-supply-chain-security.md)
- [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../../volumes/16-distributed-systems/243-message-delivery.md)
- [Chapter 256 — PHP-FPM](../../volumes/17-production-engineering/256-php-fpm.md)
- [Chapter 258 — Configuration](../../volumes/17-production-engineering/258-configuration.md)
- [Chapter 259 — Secrets](../../volumes/17-production-engineering/259-secrets.md)
- [Chapter 269 — Incident Response](../../volumes/17-production-engineering/269-incident-response.md)
- [Chapter 272 — Characterization Tests](../../volumes/18-legacy-php/272-characterization-tests.md)
- [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md)
- [Chapter 278 — PHP Version Migration](../../volumes/18-legacy-php/278-php-version-migration.md)

## Open threads

Continue Volume XVIII with Chapter 271 on legacy architecture, carrying forward the exact runtime inventory, hidden-contract evidence, security containment, and explicit migration seams established here.

## Exact next section

Chapter 271 — Legacy Architecture: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the PHP example, local Markdown links resolved, and `git diff --check` passed. Official PHP migration and history documentation was checked on 2026-09-16 for PHP 5 version boundaries, PHP 5.6-to-7.0 migration notes, deprecated functionality, and removed legacy APIs. No PHP 5 runtime, live legacy production system, database, queue, provider, or migration integration was run.

## Writing notes

Volume XVII is complete through Chapter 269. Chapter 270 begins Volume XVIII by establishing that legacy PHP work is a runtime, environment, data, deployment, and operations problem as well as a source-compatibility problem. Continue with Chapter 271 while preserving evidence-first assessment and the distinction between compatibility and safety.
