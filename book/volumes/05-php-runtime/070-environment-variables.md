---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 70
title: Environment Variables
slug: environment-variables
status: complete
summary: ../../_ai/chapter-summaries/070-environment-variables-summary.md
---

# Chapter 70 — Environment Variables

## Why This Matters

Environment variables are useful deployment inputs for database endpoints, feature switches, ports, and secret references. They are also untyped strings inherited by a process. A missing variable, an empty variable, a stale worker environment, or a value logged as a secret can become an outage or breach.

Read environment variables at the process boundary, convert them into validated configuration once, and keep `getenv()` out of business code.

## Mental Model

```text
orchestrator / shell / service manager
              ↓ inherited key=value strings
       PHP process startup
              ↓ read and validate once
       typed immutable Config object
              ↓ dependency injection
      application and worker code
```

The environment belongs to a process. CLI, FPM, tests, and queue workers may receive different values. A long-running worker keeps its startup snapshot until restart; changing a deployment secret does not mutate existing processes.

## Core Concept

`getenv()` reads an environment value. `$_ENV` is a superglobal whose population depends on PHP configuration and SAPI, and `$_SERVER` is SAPI-dependent. Use one access strategy, define missing-versus-empty semantics, and validate values. Environment values are strings.

```php
<?php

declare(strict_types=1);

function requiredEnvironment(string $name): string
{
    $value = getenv($name);
    if ($value === false || $value === '') {
        throw new RuntimeException("Missing non-empty variable: {$name}");
    }
    return $value;
}

function environmentBool(string $name, bool $default): bool
{
    $value = getenv($name);
    if ($value === false || $value === '') { return $default; }
    $parsed = filter_var($value, FILTER_VALIDATE_BOOL, FILTER_NULL_ON_FAILURE);
    if ($parsed === null) {
        throw new RuntimeException("Invalid boolean variable: {$name}");
    }
    return $parsed;
}
```

Do not use `(bool) 'false'`: every non-empty string becomes true under a naïve cast.

## Practical Example: Typed Bootstrap

```php
<?php

declare(strict_types=1);

final readonly class AppConfig
{
    public function __construct(
        public string $databaseDsn,
        public int $workerCount,
        public bool $debug,
    ) {}

    public static function fromEnvironment(): self
    {
        $workerCount = filter_var(
            getenv('WORKER_COUNT') ?: '1',
            FILTER_VALIDATE_INT,
            ['options' => ['min_range' => 1, 'max_range' => 128]],
        );
        if ($workerCount === false) {
            throw new RuntimeException('WORKER_COUNT must be 1..128.');
        }
        return new self(
            requiredEnvironment('DATABASE_DSN'),
            $workerCount,
            environmentBool('APP_DEBUG', false),
        );
    }
}
```

The application receives `AppConfig`, not the environment. Tests can construct the object without mutating global process state.

## Lifecycle and Precedence

```text
image defaults → service/container environment
                → process-manager overrides
                → bootstrap validation
                → typed config snapshot
```

Document precedence. A `.env` loader is an application/tooling convention, not the operating-system environment itself. Never let a development file silently override production-injected values.

`putenv()` changes the current process and children created afterward. It does not update already-running external processes and creates hidden mutable state. Prefer setting values before process startup. If tests use `putenv()`, isolate and restore state.

```text
new secret → new worker starts with it
          → old worker drains with old snapshot
          → supervisor stops old worker
```

## Bad Example and Better Design

```php
$timeout = (int) getenv('TIMEOUT');
if (getenv('APP_ENV') === 'production') { /* security policy */ }
```

Missing or malformed values become zero or false-like behavior, and global dependencies are hidden. Load configuration at the composition root, fail before traffic, and log only safe metadata:

```php
$config = AppConfig::fromEnvironment();
$application = new Application($config);
$application->run();
```

For secrets, prefer a secret manager or file-descriptor mechanism when available. Never include values in logs, dumps, traces, or diagnostics.

## Security

Environments can leak through process inspection, crash dumps, debug pages, support bundles, child inheritance, and logs. Limit process visibility and use safer injection mechanisms for high-value secrets when available. Validate URLs, hosts, ports, paths, and feature flags. An environment variable is deployment configuration, not proof of caller identity.

## Performance

Lookup cost is rarely a hot-path concern, but repeated global reads harm testability and consistency. Parse once during bootstrap. The operational cost is rollout: workers may need restart, and mixed versions can serve requests with different policies.

## Testing

Test missing, empty, malformed, boundary, and valid values. Construct typed config directly for unit tests. Run clean-process tests for the actual SAPI and service manager. Assert sensitive values are redacted from logs and exception output.

## Common Mistakes

- Assuming `$_ENV` is always populated.
- Treating non-empty strings as true.
- Casting malformed numbers to zero.
- Looking up variables deep in domain code.
- Expecting running workers to see changed values.
- Logging a complete environment.
- Confusing `.env` conventions with deployment security.

## Senior Engineer Thinking

Environment variables are an API between deployment and startup. Give the API a schema: required keys, grammar, defaults, ranges, precedence, redaction, rollout, and failure policy. Refusing invalid configuration at startup is easier to operate than failing on first traffic.

## Exercises

1. Build a typed loader with required, optional, boolean, integer, URL, and enum values.
2. Test missing versus explicitly empty variables.
3. Trace environment flow into CLI, FPM, and a queue worker.
4. Design rotation for workers that process jobs for several minutes.

## Review Questions

1. Why should lookup happen at the composition root?
2. What is the difference between missing and empty?
3. Why can a worker retain a stale secret?
4. What are environment-variable leak paths?
5. Which values need parsing and bounds instead of casual casts?

## Summary

Environment variables are untyped process-startup inputs. Read them through a deliberate boundary, validate them into immutable configuration, define precedence and missing-value semantics, protect secrets, and restart workers deliberately during rollout. The final chapter turns these inputs into a broader configuration design.

## References

- [PHP Manual: `getenv()`](https://www.php.net/manual/en/function.getenv.php)
- [PHP Manual: `putenv()`](https://www.php.net/manual/en/function.putenv.php)
- [PHP Manual: Predefined Variables](https://www.php.net/manual/en/language.variables.predefined.php)
- [PHP Manual: `filter_var()`](https://www.php.net/manual/en/function.filter-var.php)
- [PHP Manual: PHP configuration](https://www.php.net/manual/en/configuration.php)
