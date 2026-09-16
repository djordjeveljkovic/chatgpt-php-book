---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 258
title: Configuration
slug: configuration
status: complete
summary: ../../_ai/chapter-summaries/258-configuration-summary.md
---

# Chapter 258 — Configuration

## Why This Matters

Configuration changes how an application connects, authenticates, limits, stores, and exposes data. A missing or mis-typed value can prevent startup; a valid but unsafe value can make every request leak, overload a dependency, or disable a security control.

Treat configuration as typed input with an owner, source, precedence, validation rule, change process, and rollback. “It comes from the environment” explains transport, not semantics.

## Sources and Precedence

Common sources include:

* immutable defaults in code;
* release-specific configuration files;
* environment variables;
* mounted configuration or secret files;
* platform or service-discovery values;
* runtime feature flags.

Choose and document precedence. A local developer override should not silently beat an emergency production setting, and a stale process should not retain a value after its security or compatibility window.

Avoid a configuration hierarchy so complex that operators cannot predict the effective value. Print safe names and sources in diagnostics, never secret values.

## Parse at the Boundary

Environment variables are strings. `"false"`, `"0"`, an empty string, and an absent variable have different possible meanings. Parse once at startup into a typed object rather than scattering casts throughout request code.

~~~php
<?php

declare(strict_types=1);

final readonly class AppConfig
{
    public function __construct(
        public string $databaseDsn,
        public int $httpTimeoutMs,
        public bool $debug,
    ) {
        if ($databaseDsn === '' || $httpTimeoutMs < 1) {
            throw new InvalidArgumentException('Invalid application configuration');
        }
    }
}

function requiredEnv(array $environment, string $name): string
{
    $value = $environment[$name] ?? null;
    if (!is_string($value) || $value === '') {
        throw new RuntimeException('Missing required configuration: ' . $name);
    }

    return $value;
}

function configFrom(array $environment): AppConfig
{
    $timeout = filter_var($environment['HTTP_TIMEOUT_MS'] ?? null, FILTER_VALIDATE_INT);
    $debug = filter_var($environment['APP_DEBUG'] ?? 'false', FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE);

    if ($timeout === false || $debug === null) {
        throw new RuntimeException('Invalid configuration format');
    }

    return new AppConfig(
        requiredEnv($environment, 'DATABASE_DSN'),
        $timeout,
        $debug,
    );
}
~~~

The parser is intentionally explicit about required and optional values. Validate ranges, URLs, enum values, file paths, certificates, pool sizes, and cross-field relationships as part of startup. A config object should not contain a raw secret if the application can use a narrower secret handle.

## Fail Fast and Safe Defaults

Fail startup for missing credentials, invalid database endpoints, unsupported protocol versions, impossible resource limits, or a disabled security control that the service requires. A failed deployment is easier to detect than a running service that silently discards data.

Defaults should be safe and documented. A default development password, debug mode, wildcard origin, unlimited body size, or permissive TLS verification does not belong in a production fallback. Some values should have no default at all.

## Immutable and Dynamic Configuration

Immutable configuration is loaded at startup and changes with a release or restart. It is easy to reason about and audit, but changes require deployment or restart. Dynamic configuration can change without restart, which helps feature rollout and incident response but adds synchronization, validation, caching, and rollback complexity.

Separate safety-critical settings from experiments. A dynamic feature flag must have an owner, type, allowed values, evaluation scope, expiration, and failure behavior. Do not make a permission policy dynamic without an audit and propagation design.

## Configuration and Workers

FPM and long-running workers cache configuration in process memory. A file or environment change does not necessarily affect existing processes. Reload or restart deliberately, drain queue workers, and verify that all roles see the intended version.

If a config value changes while a job is in flight, define whether the job uses the value captured at enqueue time, the current value at execution time, or a versioned policy. Do not let a worker mix tenant or release configuration accidentally.

## Configuration as an API

Configuration names and meanings are interfaces between code, deployment, and operators. Rename or remove them with a migration window. Record effective version and source in a protected diagnostic. Test values at boundaries, not only the happy path.

Keep configuration schema close to code and review changes with their operational impact: worker count, connection pool, timeout, cache freshness, logging, security, and cost.

## Security

Configuration can contain secrets, endpoints, feature policy, and credentials. Restrict access, redact values, avoid passing secrets in command-line arguments, and do not log the complete environment. Validate allowed destinations to prevent SSRF-like configuration mistakes.

Treat configuration files and images as artifacts that require integrity and review. A malicious or stale config can be as harmful as malicious code.

## Testing

Test missing, empty, malformed, out-of-range, conflicting, deprecated, and production-insecure values. Test startup under the real service user and container limits. Verify reload and rollback, dynamic-flag propagation, worker recycling, and secret rotation.

Use a matrix for role and environment: web, worker, CLI, migration; development, staging, and production. The same name may have different safe values, but its type and invariant should remain clear.

## Common Mistakes

* Treating environment strings as typed booleans or integers without parsing.
* Hiding precedence across many files and platform layers.
* Using development defaults in production.
* Letting an invalid setting start a partially functioning service.
* Assuming an existing worker sees changed configuration immediately.
* Making feature flags permanent and ownerless.
* Logging secret values while diagnosing startup.
* Changing a config name without a migration window.

## Senior Engineer Thinking

Ask who owns each value, how it is parsed, when it takes effect, which invariant it protects, and how it rolls back. Configuration deserves the same design discipline as code because it changes runtime behavior without changing the source diff users may inspect.

## Exercises

1. Define a typed configuration schema for a web service, worker, and migration command.
2. Test the parser with absent, empty, false-looking, malformed, and out-of-range values.
3. Design a safe dynamic feature flag with owner, scope, expiry, audit, and rollback.
4. Plan configuration reload and secret rotation for FPM and long-running workers.

## Review Questions

* Why should configuration be parsed once at a boundary?
* Which settings should fail startup rather than use a default?
* What makes dynamic configuration more complex than immutable configuration?
* Why can a worker retain an old value after deployment?
* Which configuration data must not appear in logs?
* How should configuration changes be rolled back?

## Summary

Configuration is typed runtime input with sources, precedence, ownership, and lifecycle. Parse and validate it at startup, fail fast on unsafe or impossible values, choose immutable or dynamic behavior deliberately, reload workers explicitly, protect secrets and endpoints, test role/environment matrices, and record enough safe metadata to reproduce the effective runtime.

## References

- [The Twelve-Factor App: Config](https://12factor.net/config)
- [PHP manual: Environment variables](https://www.php.net/manual/en/reserved.variables.environment.php)
- [Chapter 155 — Secrets](../10-security/155-secrets.md)
- [Chapter 259 — Secrets](./259-secrets.md)
