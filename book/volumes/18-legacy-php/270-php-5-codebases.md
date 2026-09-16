---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 270
title: PHP 5 Codebases
slug: php-5-codebases
status: complete
summary: ../../_ai/chapter-summaries/270-php-5-codebases-summary.md
---

# Chapter 270 — PHP 5 Codebases

## Why This Matters

A PHP 5 codebase is not merely an old collection of syntax. It is a system shaped by an older runtime, extension set, dependency ecosystem, deployment process, security baseline, and operational culture. The same source can behave differently when moved from PHP 5.3 to 5.6, from one extension set to another, or from a short-lived Apache request to a long-running worker.

Legacy work is risky because the system often has undocumented contracts. A global variable may be populated by a server setting. A database call may depend on implicit escaping. A class may be loaded by filename conventions that are not PSR-4. A cron command may be the only writer for a table. Removing “dead” code can break a customer workflow because nobody has measured that path recently.

The first goal is not to make the code look modern. It is to understand the runtime boundary, preserve safety, establish evidence, and create a path for incremental change. PHP 5 code can often be stabilized before it is migrated, but it should not be treated as safe merely because it still runs.

## Mental Model

Assess the system across four boundaries:

~~~text
legacy source
    ↓ syntax and behavior
PHP runtime + extensions
    ↓ process and request lifecycle
application environment
    ↓ deployment and operational contracts
customer and data effects
~~~

A useful inventory records the source version assumptions, runtime version, extensions, web server and SAPI, dependency installer, database driver, scheduled jobs, queue workers, file permissions, configuration sources, secrets, and external providers. Record uncertainty rather than filling it with a guess.

PHP 5 was a family of releases, not one compatibility target. Namespaces arrived in PHP 5.3, traits and short array syntax in PHP 5.4, generators in PHP 5.5, and variadic functions in PHP 5.6. A codebase that uses namespaces may still depend on other PHP 5-era behavior. Identify the minimum and maximum runtime it actually supports.

## Runtime and Version Inventory

Start with facts:

| Boundary | Questions |
| --- | --- |
| PHP binary | Which exact version runs web and CLI code? |
| SAPI | Apache module, CGI/FastCGI, PHP-FPM, or CLI? |
| Extensions | Which are loaded, required, optional, or silently assumed? |
| Configuration | Which php.ini, environment, server variables, and includes apply? |
| Dependencies | Composer, PEAR, copied libraries, or custom autoloading? |
| Database | Driver, server version, charset, SQL mode, timezone, and transaction behavior? |
| Jobs | Which cron, queue, shell, and admin commands run, and under which user? |
| State | Where do sessions, uploads, caches, locks, and temporary files live? |
| Deployment | How are files selected, permissions changed, and processes restarted? |
| Recovery | Which artifact, backup, logs, and owner support rollback or restore? |

Inspect production configuration without printing secret values. Compare web and CLI `phpinfo`-style facts carefully; they may load different configuration and extensions. Capture command output as controlled evidence with timestamps, runtime identity, and redaction.

Do not infer runtime support from a README. Run a representative entry point under the exact binary and SAPI, or mark the result unknown.

## Common PHP 5 Compatibility Boundaries

Moving PHP 5 code to a newer runtime can expose several classes of change:

* removed or deprecated extensions and functions;
* changes to error and exception behavior;
* differences in string, array, comparison, and variable handling;
* changes to function signatures, resource types, and return values;
* changed class constructors, static calls, and inheritance assumptions;
* encoding, timezone, locale, and filesystem differences;
* extension-specific behavior such as database, XML, curl, or mcrypt support;
* code that relied on warnings, notices, or permissive coercion.

The official PHP migration guide from PHP 5.6 to PHP 7 documents backward-incompatible changes, deprecated features, changed functions, and removed extensions. Use it as a starting inventory, not as proof that a particular application is compatible. Test the application’s real paths under the target runtime.

For example, the old `mysql_*` extension was deprecated during PHP 5 and removed in PHP 7. Replacing calls mechanically is not enough: connection error handling, escaping, transaction boundaries, character sets, query behavior, and result iteration must be reviewed at the database boundary.

## Build a Safe Inventory

Search the codebase for runtime-sensitive surfaces:

* `mysql_`, `ereg`, `mcrypt`, `each`, `create_function`, and other legacy APIs;
* PHP 4-style constructors and static calls to non-static methods;
* dynamic includes, `eval`, variable variables, and string-built SQL;
* global state, superglobal mutation, and server-provided variables;
* custom session, upload, and password-handling code;
* `serialize`/`unserialize` crossing a durable or user-controlled boundary;
* shell commands, writable directories, and filesystem assumptions;
* custom autoloaders and case-sensitive filename conventions;
* cron, queue, and deployment scripts outside the main application directory.

Search results are leads, not findings. A call can be inside a dead branch, a compatibility wrapper, or a test fixture. A missing match does not prove absence when code is generated or loaded dynamically. Pair static inventory with runtime traces, configuration inspection, and representative tests.

## A Typed Legacy Finding

Use modern tooling to describe legacy risks even when the application cannot yet run on modern PHP:

~~~php
<?php

declare(strict_types=1);

enum LegacyRisk: string
{
    case Runtime = 'runtime';
    case Security = 'security';
    case Data = 'data';
    case Operations = 'operations';
    case Unknown = 'unknown';
}

final readonly class LegacyFinding
{
    public function __construct(
        public string $identifier,
        public string $location,
        public LegacyRisk $risk,
        public string $evidence,
        public string $nextCheck,
    ) {
        if ($identifier === '' || $location === '' || $evidence === '' || $nextCheck === '') {
            throw new InvalidArgumentException('Legacy finding is incomplete');
        }
    }
}

function needsImmediateContainment(LegacyFinding $finding): bool
{
    return $finding->risk === LegacyRisk::Security
        || $finding->risk === LegacyRisk::Data;
}
~~~

The catalog separates a location from the evidence and the next check. `Unknown` is valuable: it prevents a team from treating an unexecuted path as safe. A finding is not a vulnerability score or a migration plan; it is a durable prompt for investigation and ownership.

## Compatibility Layers

When one change cannot be completed at once, use a narrow compatibility layer with an expiry and owner. Examples include:

* an adapter around an old database API;
* a facade that normalizes old and new configuration names;
* a file-based loader that presents a stable interface to modern code;
* a worker boundary that translates an old message into a versioned command;
* a runtime wrapper that records missing extensions or unsupported paths.

Keep the old behavior at a named boundary. Do not spread version checks through every class. A compatibility layer should state which inputs it accepts, what behavior it preserves, what it cannot preserve, and when it will be removed.

Avoid polyfills that silently change security semantics. A wrapper that turns a failed password check into a truthy value or an unavailable random source into predictable bytes is worse than a loud startup failure.

## Security First

Legacy age increases security risk, but “modernize everything” is not a safe incident plan. Prioritize vulnerabilities and dangerous boundaries:

* unmaintained runtime and extensions;
* SQL injection, command injection, path traversal, and unsafe file uploads;
* weak password hashing, session fixation, and insecure cookies;
* untrusted deserialization and dynamic code execution;
* secrets in source, configuration, logs, or backups;
* missing authorization checks and tenant isolation;
* writable code or overly broad process permissions;
* TLS, certificate, and dependency verification assumptions.

Place controls at stable boundaries first: validate input, parameterize SQL, authorize objects, rotate secrets, isolate workers, restrict filesystem access, and record security evidence. Chapter 157 covers supply-chain risk; legacy cleanup must also protect the build and deployment path.

Do not use an unsupported PHP 5 runtime on an internet-facing system merely because migration is difficult. If a temporary containment environment is unavoidable, isolate it, restrict exposure, monitor it, and assign a dated exit plan.

## Data and Database Boundaries

Legacy code often mixes HTML, SQL, business rules, and connection state in one file. Stabilize the data boundary before changing SQL behavior. Inventory:

* connection charset and timezone;
* implicit escaping and query concatenation;
* transaction and autocommit assumptions;
* error handling and partial-write behavior;
* SQL modes, reserved words, and driver return types;
* long-running reports and batch jobs;
* triggers, stored procedures, and scheduled database tasks.

A replacement database adapter can preserve a function signature while changing error, encoding, or transaction behavior. Test business invariants, not only that a query returns rows. Chapter 277 covers detailed database migration mechanics; this chapter focuses on discovering the legacy boundary safely.

## Operational Behavior

Document how the application runs, not only how it is written:

* web requests may depend on local sessions or upload directories;
* PHP-FPM workers may load different code or configuration after a partial deployment;
* CLI jobs may run with a different working directory, user, `php.ini`, and timezone;
* cron may overlap after a slow run and repeat external effects;
* queue workers may retain process-local state indefinitely;
* log and error behavior may expose credentials or personal data;
* a shared filesystem may be a hidden coordination mechanism.

Make one safe operational change at a time. Record the exact command, binary, environment, owner, observed result, and recovery path. Chapter 269’s incident process applies when an inventory or compatibility change affects production.

## Testing a PHP 5 Codebase

Use several layers:

1. Parse and lint with the legacy runtime where possible.
2. Run static inventory and compatibility scans.
3. Execute characterization tests before changing behavior; Chapter 272 develops this in detail.
4. Test database, filesystem, mail, queue, and provider boundaries with isolated fixtures.
5. Run the application under the target runtime in a production-shaped environment.
6. Compare outputs, side effects, logs, timing, and error classification.
7. Exercise deployment, rollback, and recovery with the actual artifact and configuration.

When the PHP 5 runtime is unavailable, do not claim syntax compatibility from PHP 8 lint alone. Use an archived or isolated runtime only when it can be operated safely, and record its version, extensions, operating system, and limitations.

Test negative paths. Legacy applications often rely on notices or warnings for control flow. A missing index, invalid encoding, failed database connection, empty upload, expired session, or duplicate queue message should produce an explicit safe result.

## Performance and Capacity

Legacy performance problems often come from accidental behavior: repeated includes, unbounded queries, per-row database calls, full-file reads, session locks, or a queue worker that never releases memory. Measure before rewriting.

Record workload, PHP version, SAPI, extensions, process limits, database plan, cache state, and input size. A faster target runtime may expose a compatibility defect or change memory pressure. A code change that reduces request time can still overload the database if it increases concurrency.

Keep old and new runtimes isolated during comparison. Do not send production traffic to both versions if their writes or external effects cannot be reconciled. Use read-only or shadow paths for comparisons and disable side effects.

## Concurrency and State

Legacy code may assume one request at a time, one cron host, one web server, or one PHP process. Modern deployment can violate each assumption. Find:

* static variables and global caches;
* file locks and temporary filenames;
* session locking and shared upload names;
* singleton database connections;
* cron overlap and queue visibility leases;
* database rows used as implicit locks;
* deployment scripts that modify shared files in place.

Make ownership explicit. Use a durable lock or lease for singleton jobs, unique operation identity for retried effects, and atomic file selection for releases. Process-local state is not a replacement for a durable coordination record.

## Migration and Change Boundaries

Do not combine runtime upgrade, framework replacement, database change, and business-rule rewrite into one unmeasurable event. Separate the transitions:

~~~text
inventory and contain risk
        ↓
establish tests and observability
        ↓
introduce a stable adapter or seam
        ↓
change one boundary
        ↓
compare behavior and side effects
        ↓
remove compatibility code after evidence
~~~

Branch by abstraction, strangler migrations, characterization tests, and safe refactoring are covered in Chapters 272–275. This chapter’s rule is to preserve a known boundary and make each unknown smaller.

## Common Mistakes

* Treating “PHP 5” as one precise compatibility target.
* Upgrading the runtime before inventorying extensions and configuration.
* Replacing `mysql_*` calls without reviewing escaping, transactions, and charset.
* Running PHP 8 lint and claiming that legacy syntax works on PHP 5.
* Modernizing style before fixing security and data-boundary risks.
* Removing apparently unused globals, includes, or cron jobs without evidence.
* Testing only successful requests and ignoring warnings, notices, and partial failures.
* Running old and new workers against incompatible messages or schemas.
* Sharing legacy caches, sessions, or writable directories across runtimes.
* Keeping an emergency compatibility layer with no owner or expiry.
* Allowing an unsupported runtime to remain internet-facing indefinitely.
* Combining runtime, framework, database, and business changes into one release.

## Senior Engineer Thinking

The senior question is not “how quickly can we rewrite this?” It is “which contracts are real, which are accidental, which risks are urgent, what evidence is missing, and how can we create a safer seam for the next change?”

Legacy engineering is archaeology with production consequences. Respect the system’s observed behavior without treating every behavior as desirable. Preserve valuable contracts, replace dangerous accidents, make uncertainty visible, and improve the runtime and operational boundary before pursuing cosmetic modernization.

## Exercises

1. Build a PHP 5 runtime inventory covering version, SAPI, extensions, configuration, dependencies, database, jobs, state, deployment, and recovery.
2. Create ten LegacyFinding records from a real or sample codebase. Classify risk, evidence, next check, owner, and containment.
3. Choose one legacy database API and design an adapter that preserves explicit error, charset, transaction, and parameter contracts.
4. Compare a legacy application under its current runtime and a target runtime using read-only paths. List differences in outputs, warnings, timing, memory, and side effects.

## Review Questions

* Why is a PHP 5 codebase more than old syntax?
* Which facts must be inventoried before changing its runtime?
* Why can PHP 5.3 and PHP 5.6 be different compatibility targets?
* Why are search results leads rather than proof of a runtime finding?
* What makes a compatibility layer safe to keep temporarily?
* Which security boundaries should be stabilized first?
* Why must database behavior be tested beyond returned rows?
* What operational assumptions can PHP-FPM, cron, and queue workers violate?
* Why is PHP 8 lint insufficient to prove PHP 5 compatibility?
* How do explicit seams make legacy change safer?

## Summary

A PHP 5 codebase is an old runtime, dependency, application, data, deployment, and operational system—not merely old syntax. Inventory exact versions and hidden contracts, distinguish evidence from assumption, stabilize security and data boundaries, isolate unsupported runtimes, use narrow expiring compatibility layers, test real behavior under each target environment, and separate changes into measurable transitions. Legacy work becomes safer when uncertainty is cataloged, risky behavior is contained, and each new seam preserves a verifiable contract.

## References

- [PHP Manual: Migrating from PHP 5.6.x to PHP 7.0.x](https://www.php.net/migration70)
- [PHP Manual: Deprecated features in PHP 7.0.x](https://www.php.net/manual/en/migration70.deprecated.php)
- [PHP RFC: Remove deprecated functionality in PHP 7](https://wiki.php.net/rfc/remove_deprecated_functionality_in_php7)
- [PHP Manual: History of PHP](https://www.php.net/manual/en/history.php.php)
- [Chapter 92 — Composer](../07-composer-and-the-php-ecosystem/092-composer.md)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 256 — PHP-FPM](../17-production-engineering/256-php-fpm.md)
- [Chapter 258 — Configuration](../17-production-engineering/258-configuration.md)
- [Chapter 259 — Secrets](../17-production-engineering/259-secrets.md)
- [Chapter 269 — Incident Response](../17-production-engineering/269-incident-response.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 277 — Database Migration](./277-database-migration.md)
- [Chapter 278 — PHP Version Migration](./278-php-version-migration.md)
