---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 278
title: PHP Version Migration
slug: php-version-migration
status: complete
summary: ../../_ai/chapter-summaries/278-php-version-migration-summary.md
---

# Chapter 278 — PHP Version Migration

## Why This Matters

Changing the PHP version changes the language, runtime, extensions, configuration, process behavior, dependency platform, and sometimes the database driver. A source tree that works on PHP 5.6 is not automatically compatible with PHP 7 or PHP 8, and “PHP 5” itself describes several materially different targets.

A safe migration is an evidence-driven compatibility transition. Identify the exact source and target environments, scan for known changes, run the application’s real paths, compare behavior and resource use, roll out in stages, and keep a recovery path honest. A successful build is necessary; it is not proof that all entry points, extensions, workers, and data contracts are safe.

## Mental Model

Treat a runtime migration as a matrix, not a version string:

~~~text
source code + dependencies
          × PHP binary + extensions + php.ini
          × SAPI + process lifecycle
          × database driver + SQL mode
          × OS + filesystem + locale/timezone
          × entry point + workload
~~~

The target is compatible only for the tested combinations. Web and CLI can load different configuration. PHP-FPM workers can retain code and process-local state. A queue worker may run an old artifact after web traffic has moved. An extension may be installed for one SAPI but missing from another.

## Define the Migration Contract

Write the contract before changing the runtime:

~~~text
source: PHP 5.6.40, FPM and CLI, required extensions A/B/C
target: PHP 8.3, FPM and CLI, replacement for extension B
preserved: authorization, money precision, response/error contract
changed intentionally: warning handling and unsupported API removal
entry points: web, CLI, cron, queue, admin, repair
evidence: compatibility scan, characterization, integration, capacity
rollback: old artifact while schema/messages remain compatible
stop: security, data, effect, or fatal-error regression
~~~

Chapter 270 covers the inventory. Chapter 277 covers database compatibility. This chapter turns them into a version-transition contract.

## Choose the Target Deliberately

Do not choose a target only because it is the newest binary available. Consider:

* security support and patch availability;
* application and framework requirements;
* Composer platform constraints and lockfile resolution;
* extension replacements and native APIs;
* database driver and client-library behavior;
* SAPI, web server, FPM, CLI, and worker support;
* memory, CPU, startup, and concurrency changes;
* deployment image, operating system, and observability;
* rollback compatibility with schema, messages, sessions, and caches.

The target must be specific: major/minor/patch, extension list, configuration, SAPI, operating system, and deployment artifact. “PHP 8” is insufficient for a high-risk compatibility claim.

## Version and API Boundaries

The PHP migration guide from PHP 5.6 to PHP 7 documents backward-incompatible changes, deprecated features, changed functions, and removed extensions. Use the [official migration guide](https://www.php.net/migration70) as a checklist, not as an application test.

Common categories include:

* syntax that the target parser rejects or interprets differently;
* removed functions, extensions, or resource types;
* changed parameter, return, comparison, string, array, or error behavior;
* warnings or notices that become exceptions or fatal errors;
* changed inheritance, constructors, visibility, or static-call rules;
* changed serialization, JSON, date/time, locale, or encoding behavior;
* configuration directives and default changes;
* extension APIs whose replacement has different precision or failure semantics.

The [PHP history documentation](https://www.php.net/manual/en/history.php.php) reinforces that PHP 5 was a series of releases, not one identical runtime. Verify exact claims against the target version’s migration documentation and test the application paths that exercise them.

## Compatibility Scan

Scan source, dependencies, configuration, deployment, and jobs for:

* removed or deprecated functions and extensions;
* PHP 4-style constructors, dynamic properties, and static calls;
* variable variables, dynamic includes, `eval`, and generated code;
* assumptions about `false`, `null`, numeric strings, array keys, or warnings;
* custom error handlers, shutdown handlers, and output buffering;
* serialized class names, session data, queue messages, and cache values;
* Composer constraints, plugins, platform requirements, and lock files;
* `php.ini`, FPM pool settings, CLI wrappers, cron, workers, and shell scripts.

Search results are leads. A deprecated call in a dead branch is different from one used for authentication or payment. A missing extension may be hidden behind an adapter. Classify findings by reachability, consequence, evidence, and next check.

## A Compatibility Gate

Represent a release gate with explicit evidence:

~~~php
<?php

declare(strict_types=1);

enum CompatibilityStatus: string
{
    case Pass = 'pass';
    case Investigate = 'investigate';
    case Block = 'block';
}

final readonly class CompatibilityCheck
{
    public function __construct(
        public string $name,
        public CompatibilityStatus $status,
        public string $evidence,
        public string $environment,
    ) {
        if ($name === '' || $evidence === '' || $environment === '') {
            throw new InvalidArgumentException('Compatibility check is incomplete');
        }
    }
}

function mayRelease(array $checks): bool
{
    foreach ($checks as $check) {
        if (!$check instanceof CompatibilityCheck) {
            throw new InvalidArgumentException('Unexpected compatibility check');
        }

        if ($check->status === CompatibilityStatus::Block) {
            return false;
        }
    }

    return $checks !== [];
}
~~~

`Investigate` is not a pass. The gate permits a release only when no check blocks it, but the release policy must also define which investigations are acceptable, who approves them, and which evidence is required for each capability. Security, data, authorization, and unknown-completion findings should normally block the relevant rollout.

## Dependencies and Extensions

Composer platform requirements can express PHP and extension constraints, but resolution is not runtime proof. Confirm the actual binary and loaded modules in the build and in every production SAPI. A lockfile selected under one platform may not install or behave identically under another.

For each extension:

1. identify direct and transitive consumers;
2. record the API, return types, error behavior, and precision relied upon;
3. choose a replacement or isolate the adapter;
4. test web, CLI, cron, and workers separately;
5. remove the old dependency only after all consumers are migrated.

Do not replace `mcrypt`, database APIs, XML libraries, or random-number sources with a similarly named package without reviewing security and semantic differences. Chapter 157 covers dependency and supply-chain controls.

## Configuration and SAPI

Compare source and target values for:

* loaded `php.ini` and additional scanned files;
* extensions and extension versions;
* memory, execution, input, upload, and post limits;
* error reporting, display, logging, and shutdown behavior;
* timezone, locale, encoding, and default character handling;
* OPcache, FPM pools, environment variables, and file permissions;
* CLI wrappers, working directory, user, and process limits.

Run an identity check from each entry point and redact secrets. Do not assume a container image’s CLI matches its FPM process. Chapter 256 covers FPM and Chapter 258 covers configuration.

## Characterization and Compatibility Testing

Use Chapter 272’s observations and run them under the source and target environments:

~~~text
same fixture + source runtime → normalized observation A
same fixture + target runtime → normalized observation B
                                      ↓
                         classify every difference
~~~

Test:

* parser and startup behavior;
* authentication, authorization, and tenant isolation;
* money, dates, encoding, sorting, and pagination;
* database queries, transactions, locks, and affected rows;
* files, uploads, sessions, caches, mail, queues, and webhooks;
* warnings, exceptions, response codes, headers, and exit codes;
* retries, duplicate messages, worker restarts, and partial effects;
* memory, latency, query count, lock duration, and throughput.

PHP 8.2+ linting can validate the modern example or adapter; it cannot prove that PHP 5 parses or executes the legacy source. Run the legacy application on its exact supported runtime when that compatibility claim matters. If the runtime is unavailable, report the result as partial evidence.

## Runtime Compatibility Adapter

Keep version checks at a boundary:

~~~text
legacy caller → compatibility adapter → target API
                         ↓
                    evidence + expiry
~~~

The adapter should document accepted inputs, preserved behavior, changed behavior, failure mapping, owner, and removal condition. Do not scatter `if (PHP_VERSION_ID < ...)` through business code. Do not make a security-sensitive fallback silent; fail closed when the replacement cannot provide equivalent guarantees.

## Staged Rollout

A runtime rollout should progress independently for web and asynchronous processes:

1. build and scan the target artifact;
2. run startup and identity checks in an isolated environment;
3. execute characterization and integration suites;
4. run read-only, internal, or synthetic traffic;
5. upgrade one low-risk web cohort;
6. upgrade CLI and cron with explicit schedules and locks;
7. drain and upgrade queue workers with message compatibility;
8. expand by failure domain while observing capacity and errors;
9. retire the old artifact after recovery and compatibility evidence.

Do not upgrade web workers and queue consumers together if their rollback or message contracts differ. A worker may hold old code while a web request writes new message fields. Chapter 264 covers deployment; Chapter 265 covers rollback.

## Performance and Resource Changes

Measure before and after with the same workload and environment shape:

* request and job latency distributions;
* CPU, memory, process count, and worker recycling;
* database connections, query plans, rows, and lock waits;
* cache, session, queue, and provider behavior;
* startup time and deployment drain duration;
* error, warning, retry, and effect rates.

A target runtime that lowers per-request CPU can still increase database concurrency if FPM capacity is raised. A target that changes memory behavior may require worker recycling or a smaller pool. Chapter 223 covers performance measurement and Chapter 235 covers capacity.

## Rollback and Recovery

Runtime rollback is safe only while the old artifact can read and write the resulting state. Before rollout, verify:

* schema and status compatibility;
* session, cache, and serialized message compatibility;
* old and new workers’ acknowledgment and retry behavior;
* external effects and operation identity;
* deployment artifact availability and configuration provenance;
* database backup and forward-recovery paths.

If the target runtime has committed new data or emitted effects, restore the old binary only after compatibility and reconciliation checks. A runtime rollback cannot undo a payment, email, message, file publication, or schema change. Chapter 267 covers backup and restore evidence.

## Common Mistakes

* Treating PHP 5.3, 5.4, 5.5, and 5.6 as one target.
* Choosing “PHP 8” without a patch, SAPI, extension, and configuration matrix.
* Treating Composer resolution or PHP 8.2+ lint as runtime compatibility proof.
* Scanning source while ignoring config, jobs, workers, extensions, and serialized data.
* Replacing an extension mechanically without checking precision, errors, and security.
* Testing web requests but not CLI, cron, queue, or repair paths.
* Accepting warnings, changed comparisons, or output differences without classification.
* Upgrading workers before message and retry compatibility is established.
* Increasing concurrency because the new runtime is faster without checking database capacity.
* Falling back silently to a weaker security implementation.
* Rolling back the binary after irreversible data or external effects.
* Keeping compatibility checks and version branches without owners or expiry.

## Senior Engineer Thinking

The senior question is not “does the new PHP binary start?” It is “which exact environment and entry points have been proven, which behavior changes are intentional, what extension and data contracts remain, and what happens after a partial rollout?”

Runtime migration is a systems change. Keep source, dependency, SAPI, configuration, process, database, workload, and recovery evidence together. Roll out the smallest meaningful boundary, make unknowns visible, and prefer a supported target with a tested path over an apparently simple upgrade that cannot be recovered honestly.

## Exercises

1. Build a source/target compatibility matrix covering PHP patch, SAPI, extensions, `php.ini`, database driver, OS, and entry point.
2. Create ten `CompatibilityCheck` values from a legacy application. Classify each as pass, investigate, or block and attach evidence.
3. Choose one removed or deprecated extension. Design an adapter and list API, precision, errors, security, and worker-lifecycle behavior to test.
4. Write a staged rollout for FPM, CLI, cron, and queue workers. Include message compatibility, capacity, stop conditions, and rollback.
5. Describe a runtime rollback after a new worker has written data and emitted a message. Separate code rollback, data repair, and effect recovery.

## Review Questions

* Why is a PHP version migration a matrix rather than a version string?
* Which parts of the source and target environments must be specified?
* Why are PHP 5 minor versions distinct compatibility targets?
* What can compatibility scans find, and what can they miss?
* Why do Composer platform requirements not prove runtime behavior?
* How should web, CLI, cron, and queue processes be tested separately?
* What does `Investigate` mean in a compatibility gate?
* Why must version checks live at boundaries rather than in business code?
* Which resource changes can occur even when output matches?
* When is runtime rollback unsafe, and what forward recovery may be required?

## Summary

PHP version migration changes source compatibility, extensions, dependencies, configuration, SAPI, process lifetime, database drivers, and resource behavior. Define an exact source/target matrix and migration contract, scan APIs and operational surfaces, replace extensions deliberately, verify every SAPI and entry point, compare characterization behavior and capacity, keep compatibility adapters narrow, roll out web and workers in controlled stages, and treat rollback as a schema/data/effect decision. A passing build is evidence of one boundary; it is not proof of whole-system compatibility.

## References

- [PHP Manual: Migrating from PHP 5.6.x to PHP 7.0.x](https://www.php.net/migration70)
- [PHP Manual: Deprecated features in PHP 7.0.x](https://www.php.net/manual/en/migration70.deprecated.php)
- [PHP Manual: History of PHP](https://www.php.net/manual/en/history.php.php)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 223 — Performance Mental Model](../15-performance/223-performance-mental-model.md)
- [Chapter 235 — Scaling](../15-performance/235-scaling.md)
- [Chapter 256 — PHP-FPM](../17-production-engineering/256-php-fpm.md)
- [Chapter 258 — Configuration](../17-production-engineering/258-configuration.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 267 — Backups](../17-production-engineering/267-backups.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 274 — Strangler Pattern](./274-strangler-pattern.md)
- [Chapter 277 — Database Migration](./277-database-migration.md)
