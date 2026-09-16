---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 71
title: Configuration
slug: configuration
status: complete
summary: ../../_ai/chapter-summaries/071-configuration-summary.md
---

# Chapter 71 — Configuration

## Why This Matters

Configuration decides how a PHP process parses requests, allocates resources, reports errors, connects to dependencies, and behaves in each environment. A bad setting can be as damaging as bad code: an unlimited upload, production error display, a short timeout, or a worker that never reloads a changed secret.

Configuration is not a bag of constants. It is versioned, typed input to a process with ownership, precedence, validation, and rollout semantics.

## Mental Model

```text
PHP binary defaults
       ↓ php.ini / scanned INI / SAPI settings
process environment and secret sources
       ↓ application bootstrap
validated immutable application config
       ↓ dependency construction
runtime behavior and feature policy
```

Keep distinct:

1. PHP runtime configuration: interpreter, extensions, limits, and SAPI behavior.
2. Deployment configuration: endpoints, credentials, resources, and process-manager values.
3. Application configuration: domain and product policy in typed structures.

## Core Concept: Scope and Mutability

PHP INI directives have changeability modes documented by PHP. Some are changeable only in system or per-directory configuration, some at runtime, and some effectively require startup. `ini_get()` reads the current value; `ini_set()` attempts a process/request-level change where permitted. A changed value is not necessarily shared by every SAPI or future worker.

```php
<?php

declare(strict_types=1);

$previous = ini_set('display_errors', '0');
if ($previous === false) {
    throw new RuntimeException('display_errors could not be changed here.');
}
$memoryLimit = ini_get('memory_limit');
if ($memoryLimit === false) {
    throw new RuntimeException('memory_limit is unavailable.');
}
```

Do not use runtime mutation as a substitute for reviewed deployment configuration. It can make behavior depend on request order or bootstrap path.

## Configuration Schema

A schema answers:

```text
name → source → type → default → allowed range → secret status → reload policy
```

| Key | Type | Rule | Reload policy |
| --- | --- | --- | --- |
| database DSN | structured string | required, approved scheme | restart/pool reload |
| request timeout | positive duration | bounded | deploy-controlled |
| debug | boolean/enum | false in production | restart |
| upload limit | bytes | below storage quota | deploy-controlled |
| feature flag | enum | explicit fallback | possibly dynamic |

A default is a product decision. A missing payment endpoint should fail startup; a missing optional metrics label may have a default.

## Minimal Example: Validated Config

```php
<?php

declare(strict_types=1);

final readonly class RuntimeConfig
{
    public function __construct(
        public string $environment,
        public int $requestTimeoutSeconds,
        public bool $debug,
        public string $databaseDsn,
    ) {}

    public static function from(array $input): self
    {
        $environment = $input['environment'] ?? null;
        $timeout = $input['request_timeout_seconds'] ?? null;
        $debug = $input['debug'] ?? null;
        $dsn = $input['database_dsn'] ?? null;
        if (! is_string($environment) || ! in_array($environment, ['development', 'test', 'production'], true)) {
            throw new InvalidArgumentException('Invalid environment.');
        }
        if (! is_int($timeout) || $timeout < 1 || $timeout > 300) {
            throw new InvalidArgumentException('Invalid request timeout.');
        }
        if (! is_bool($debug)) {
            throw new InvalidArgumentException('Debug must be boolean.');
        }
        if (! is_string($dsn) || $dsn === '') {
            throw new InvalidArgumentException('Database DSN is required.');
        }
        if ($environment === 'production' && $debug) {
            throw new InvalidArgumentException('Debug cannot be enabled in production.');
        }
        return new self($environment, $timeout, $debug, $dsn);
    }
}
```

The object is immutable. In a long-running process, dependencies see one snapshot; reload becomes an explicit replacement of the process or dependency graph rather than hidden mutation.

## Files, INI, and Environment Sources

`parse_ini_file()` reads an INI file, but its syntax and value interpretation are not automatically an application schema. Use it for a bounded format, validate its result, and keep secrets out of source-controlled files. PHP’s own INI syntax and directive semantics differ from a generic config format.

A clear precedence policy might be:

```text
safe code defaults < checked-in environment defaults
                  < deployment environment variables
                  < approved runtime overrides
```

Do not let request parameters override runtime policy. Avoid silent deep merges where a collision could change authorization, storage, or network behavior. Emit a configuration version or checksum, but redact secret values.

## Lifecycle Diagram: Startup and Reload

```text
process starts → read INI/SAPI and environment
              → load application sources
              → validate cross-field invariants
              ├─ invalid → safe diagnostics → exit
              └─ valid → construct dependencies → serve

configuration change → reload policy → new process snapshot
                     → old process drains and exits
```

For FPM, changed files or environment values may not affect existing workers until the relevant reload/restart. CLI naturally starts a new process. Worker reload must be designed with Chapter 66 signal handling.

## Bad Example and Better Design

```php
$timeout = $_GET['timeout'] ?? ini_get('default_socket_timeout');
ini_set('display_errors', $_GET['debug'] ?? '0');
```

This lets request input control process policy and leaves types undefined. Instead load at the composition root and inject explicit dependencies:

```php
$raw = [
    'environment' => requiredEnvironment('APP_ENV'),
    'request_timeout_seconds' => parsePositiveInt(requiredEnvironment('REQUEST_TIMEOUT_SECONDS')),
    'debug' => environmentBool('APP_DEBUG', false),
    'database_dsn' => requiredEnvironment('DATABASE_DSN'),
];
$config = RuntimeConfig::from($raw);
$application = new Application($config);
```

The parser should reject malformed numbers rather than quietly converting them to zero.

## Failure and Observability

Configuration errors should be early, specific, and safe:

```text
missing key → identify key name; never print secret value
bad type    → show expected grammar and source
conflict    → name the incompatible keys
dependency  → distinguish unavailable from malformed
```

Log a configuration version, environment, and non-secret capabilities. Never dump the merged configuration. Distinguish “process alive” from “configuration and dependencies ready.”

## Performance

Parsing and validating once is cheap compared with repeatedly reading files, environment, or remote stores during requests. Caching compiled configuration can reduce startup work but introduces invalidation risk. Dynamic flags are distributed reads, not constants. Measure effective settings in the target SAPI; CLI defaults do not automatically predict FPM.

## Security

Use secure defaults, least privilege, and fail-closed behavior for security-sensitive settings. Keep secrets out of Git, diagnostics, exception messages, and client responses. Review network destinations, certificate verification, debug settings, file paths, and command binaries as policy. Protect config files and the deployment pipeline.

## Database Interaction

Database configuration includes credentials, host, pool limits, timeouts, transaction behavior, and migration policy. Validate connection options before traffic, but avoid a startup health check that creates a thundering herd during recovery. Connection creation needs a deadline and readiness signal. Do not hide destructive schema migrations in ordinary request bootstrap.

## Testing

Test schema defaults, invalid values, boundaries, cross-field rules, and redaction as pure functions. Add a matrix for CLI, FPM, workers, and production-like settings. Smoke-test the real INI and environment, verify effective values through safe diagnostics, and test reload/rollback with old workers draining on their old snapshot.

## Common Mistakes

- Treating all configuration sources as interchangeable.
- Using `ini_set()` for values that must be consistent across workers.
- Starting with invalid configuration and failing on first traffic.
- Using strings and magic defaults throughout the application.
- Caching without an invalidation plan.
- Printing merged configuration during an incident.
- Assuming CLI and FPM have the same effective INI values.

## Senior Engineer Thinking

Configuration is an interface with compatibility requirements. Changes need ownership, review, versioning, rollout, rollback, and observability like code changes. The best configuration object makes invalid states difficult to construct and operational decisions visible.

## Exercises

1. Define a schema for a file-import worker with paths, byte limits, batch size, timeout, and retries.
2. Implement a redacted configuration report.
3. Compare `php --ini`, `ini_get()`, and environment inspection in CLI and FPM-like tests.
4. Design zero-downtime configuration rollout with validation, new workers, draining, and rollback.

## Review Questions

1. How do PHP INI and application configuration differ?
2. Why validate before serving traffic?
3. What does an immutable snapshot buy a worker?
4. Which changes require restart or reload?
5. How can diagnostics remain useful without exposing secrets?

## Summary

Configuration is typed, scoped, versioned input to a PHP process. Separate runtime settings, deployment inputs, and application policy; define precedence; validate cross-field invariants at startup; inject an immutable snapshot; and design reload, rollback, secret handling, and observability deliberately. This completes Volume 5’s runtime boundary.

## References

- [PHP Manual: Configuration](https://www.php.net/manual/en/configuration.php)
- [PHP Manual: The configuration file](https://www.php.net/manual/en/configuration.file.php)
- [PHP Manual: `ini_get()`](https://www.php.net/manual/en/function.ini-get.php)
- [PHP Manual: `ini_set()`](https://www.php.net/manual/en/function.ini-set.php)
- [PHP Manual: `parse_ini_file()`](https://www.php.net/manual/en/function.parse-ini-file.php)
- [PHP Manual: INI directives](https://www.php.net/manual/en/ini.list.php)
