---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 60
title: PHP CLI
slug: php-cli
status: complete
summary: ../../_ai/chapter-summaries/060-php-cli-summary.md
---

# Chapter 60 — PHP CLI

## Why This Matters

The command line is PHP's simplest visible runtime boundary. There is no browser, reverse proxy, or FastCGI peer to hide the process: a shell starts `php`, PHP reads arguments and standard input, the script runs, and an exit status returns to the shell. That simplicity makes CLI PHP useful for learning and for production work such as migrations, scheduled commands, queue consumers, deployment hooks, and diagnostics.

It is also a source of incorrect assumptions. A script that works from a terminal may fail under cron, systemd, a container, or a queue supervisor because the working directory, environment, user, configuration, input streams, and timeout policy differ. CLI PHP uses the PHP language, but it is a different SAPI and a different operating contract from PHP-FPM.

```text
shell / supervisor
        ↓ argv, environment, cwd, stdin
php CLI SAPI
        ↓ startup, configuration, compile, execute
PHP program
        ↓ stdout, stderr, exit status, files/database/network
shell / supervisor / pipeline
```

The useful question is not “does this command run locally?” It is “what inputs and guarantees does this invocation have, and what does its exit status tell its caller?”

## Mental Model

One CLI invocation is normally one operating-system process:

```text
process starts
  → CLI startup and ini loading
  → script is compiled
  → top-level code executes
  → shutdown functions and request teardown run
  → process exits with a status code
```

The process has three conventional streams:

| Stream | Intended use | Typical consumer |
| --- | --- | --- |
| `STDIN` | Input data or commands | Pipe, terminal, file |
| `STDOUT` | The command's successful result | User, pipe, capture file |
| `STDERR` | Diagnostics and progress | Terminal, log collector |

Keep data and diagnostics separate. If a command emits JSON on standard output, a progress bar on the same stream corrupts the JSON. In PHP, `STDIN`, `STDOUT`, and `STDERR` are available in CLI scripts; code intended for several SAPIs can use `defined('STDOUT')` or an injected output abstraction instead of assuming those constants exist.

## Core Concept

### Invocation is part of the API

The shell supplies arguments as strings. PHP exposes the argument count and vector through `$argc` and `$argv` in CLI execution:

```php
<?php

declare(strict_types=1);

$command = $argv[1] ?? null;

if ($command === null) {
    fwrite(STDERR, "Usage: php report.php <date>\n");
    exit(64); // EX_USAGE on systems that use sysexits values.
}

fwrite(STDOUT, "Generating report for {$command}\n");
```

The exact argument syntax is your command contract. Validate it before opening files or changing data. A value such as `--limit=0`, an absent value, or a repeated option should have an intentional meaning rather than being silently coerced.

For anything beyond a tiny script, use a command parser or a small application-level parser that returns a typed command object. Do not let business code read `$argv` throughout the codebase. The adapter translates strings into application input; the application remains reusable from HTTP, a test, or another command.

### Configuration is selected before application code

The CLI binary can load a configured `php.ini`, accept additional settings with `-d`, use a specific file with `-c`, or ignore configuration with `-n`. The effective configuration depends on the binary, environment, command-line flags, and loaded extensions:

```bash
php --ini
php -i
php -r 'echo PHP_SAPI, " ", PHP_VERSION, PHP_EOL;'
php -d memory_limit=512M bin/import.php --source=data.csv
```

These commands answer different questions. `php --ini` shows configuration-file selection; `php -i` reports runtime information; `-r` runs a short expression without a script file. A production command should make its PHP binary and configuration discoverable, rather than relying on whichever `php` appears first in an interactive `PATH`.

### Exit status is a machine-readable result

A command communicates more than text. Its exit status is how a scheduler or supervisor decides whether the invocation succeeded:

```php
<?php

declare(strict_types=1);

try {
    runImport();
    fwrite(STDOUT, "import complete\n");
    exit(0);
} catch (Throwable $exception) {
    fwrite(STDERR, $exception->getMessage() . PHP_EOL);
    exit(1);
}
```

Do not catch `Throwable` merely to hide failures. Catch it at the outer command boundary when you can add a useful diagnostic and return a stable status; log the exception with context and preserve the failure for the supervisor. A successful exit should mean the promised work completed, not merely that the process reached its last line.

## How It Works

At startup the CLI SAPI initializes PHP, reads applicable configuration, loads extensions, and creates the execution context for the script. The engine then compiles and executes the entry file and anything it includes or autoloads. The same language/runtime concepts discussed in Chapters 40–59 apply, but the SAPI supplies different inputs and output behavior.

The command-line options are documented in the [PHP Manual command-line options](https://www.php.net/manual/en/features.commandline.options.php). Useful categories include:

- `-f` to execute a file explicitly;
- `-r` to execute code without PHP opening tags;
- `-l` to lint without executing the file;
- `-d name=value` to set an ini value for this invocation;
- `-c` and `-n` to control configuration loading;
- `-v`, `-m`, and `--ini` to inspect the runtime;
- `-S` to start the built-in development server.

The built-in server is valuable for local development and demonstrations. It is not a general production front end; production needs a deliberately configured web server and application runtime.

### Standard input and pipes

A command can accept a stream without first materializing a whole file:

```php
<?php

declare(strict_types=1);

$count = 0;

while (($line = fgets(STDIN)) !== false) {
    $line = trim($line);

    if ($line === '') {
        continue;
    }

    processRecord($line);
    $count++;
}

if (!feof(STDIN)) {
    fwrite(STDERR, "Unable to read all input\n");
    exit(1);
}

fwrite(STDERR, "processed={$count}\n");
```

This is an O(1)-application-memory shape with respect to the number of lines, assuming `processRecord()` does not retain them. It also means partial input is possible: a pipe can close or an upstream command can fail. Decide whether the operation is restartable, resumable, or transactional before processing irreversible records.

## Practical Example: A Safe Import Command

Separate the adapter, work unit, and result:

```php
<?php

declare(strict_types=1);

final readonly class ImportOptions
{
    public function __construct(
        public string $path,
        public int $batchSize,
    ) {
    }
}

function parseOptions(array $arguments): ImportOptions
{
    $path = $arguments[1] ?? '';
    $batchSize = isset($arguments[2]) ? filter_var($arguments[2], FILTER_VALIDATE_INT) : 100;

    if ($path === '' || $batchSize === false || $batchSize < 1) {
        throw new InvalidArgumentException('Usage: php import.php <path> [batch-size]');
    }

    return new ImportOptions($path, $batchSize);
}

try {
    $options = parseOptions($argv);
    $processed = importFile($options);
    fwrite(STDOUT, "processed={$processed}\n");
} catch (InvalidArgumentException $exception) {
    fwrite(STDERR, $exception->getMessage() . PHP_EOL);
    exit(64);
} catch (Throwable $exception) {
    error_log($exception->__toString());
    fwrite(STDERR, "Import failed; see the log for details.\n");
    exit(1);
}
```

The import itself should define its transaction and restart policy. A batch size is not automatically a transaction boundary: committing each batch improves lock duration and recovery cost, but a crash can leave earlier batches committed. Use an idempotency key, a durable cursor, or a staging table when reruns must be safe.

## Production Example: Scheduling

Cron, systemd timers, containers, and external schedulers all invoke commands differently. Make the command independent of the interactive shell:

```bash
/usr/bin/php /srv/releases/2026-09-14/bin/rebuild-report.php \
  --date=2026-09-13 \
  >>/var/log/app/rebuild-report.log 2>&1
```

Use absolute paths when the deployment contract requires them. Set the working directory explicitly if relative paths are part of the command, or resolve paths from `__DIR__` and configuration. Prevent overlapping runs with a scheduler facility or a distributed/OS lock whose failure semantics are understood. A lock that never expires can block every future run; a lock that expires too early permits concurrent writers.

## Failure Modes

### “It works in my terminal”

The terminal may provide a different `PATH`, locale, current directory, home directory, PHP binary, ini file, extensions, and credentials than the scheduler. Record safe startup facts such as `PHP_SAPI`, PHP version, release identifier, and configuration location. Never dump secrets into logs merely to diagnose environment differences.

### Partial progress and retries

A process can die after writing 900 of 1,000 records. A scheduler may retry the command, so “retry on non-zero exit” is only safe if the work is idempotent or resumable. Store a stable source identifier and use database uniqueness or an upsert policy where appropriate. Do not use the process exit code as a substitute for durable progress.

### Signals and termination

Ctrl-C, a container stop, a supervisor restart, and an out-of-memory kill are different events. A command may receive a signal while waiting on I/O, and a hard kill provides no cleanup opportunity. Signal handling and graceful shutdown are developed in Chapter 66; design this command so an interrupted invocation can be safely retried even if cleanup does not run.

### Shell boundaries

Passing untrusted data to `shell_exec()`, `exec()`, or a constructed shell command can become command injection. Prefer PHP libraries and direct file/database APIs. When a process is genuinely required, pass a fixed executable and carefully escaped arguments, constrain environment and working directory, and test hostile values. `escapeshellarg()` reduces one class of argument injection; it does not turn an arbitrary shell pipeline into a safe design.

## Performance

CLI startup cost matters for millions of tiny invocations, while throughput and memory usually dominate imports and workers. Measure wall time, CPU, peak memory, records per second, database round trips, and external calls. Avoid a loop that performs one query or one process spawn per record when batching is valid.

Streaming input limits application memory, but the database may still retain locks or transaction state. Batch commits bound recovery work and lock duration at the cost of more commits and partial progress. OPcache can be enabled for CLI with the relevant configuration, but a warm web worker and a fresh CLI process have different startup paths; benchmark the deployed SAPI.

## Security

CLI is not automatically trusted. Arguments, standard input, environment variables, files, and database rows can all be attacker-controlled. Run commands with the least-privileged user that can perform the job. Avoid writing secrets in command arguments when process listings or scheduler logs can expose them; prefer protected environment/configuration mechanisms or a secret store.

Validate paths and reject unexpected schemes or traversal where a user can influence a file name. Use exclusive file creation and correct permissions for temporary files. Redact tokens, passwords, and personal data from progress output. A failed command must not print a full exception containing credentials to a shared terminal or log.

## Testing

Test the application command without making the test suite depend on a real terminal:

1. Unit-test argument parsing for missing, repeated, malformed, boundary, and unknown options.
2. Test the command runner with injected input/output streams and a fake importer.
3. Integration-test database transactions, uniqueness, batching, and restart behavior.
4. Run a subprocess test that verifies stdout, stderr, and exit status together.
5. Exercise the exact container or systemd/cron invocation in a smoke test.

Keep a fixture that fails halfway through an import. Run it twice and assert the promised idempotency or resume behavior. Also test a closed or malformed stdin stream; a command that only works with an interactive terminal is not a robust pipeline component.

## Exercises

1. Write a command that reads newline-delimited identifiers from `STDIN`, writes a count to `STDOUT`, and writes progress to `STDERR`. Pipe its output into another program and verify that the data remains machine-readable.
2. Predict the behavior of a command invoked from a different working directory. Refactor relative file access so the behavior is explicit.
3. Design a restart-safe import with batches of 100 records. State the invariant that makes a retry safe and the database constraint that supports it.
4. Run `php --ini`, `php -m`, and a `-d` override under the same user as the scheduler. Document the differences you would alert on.

## Review Questions

1. What is the contract of each standard stream, and why should diagnostics usually use `STDERR`?
2. Why is a zero exit status not enough evidence that a command fulfilled its business promise?
3. Which environment differences commonly explain a cron failure that cannot be reproduced in a shell?
4. Why can streaming reduce PHP memory without making a database transaction safe to retry?
5. What makes an import idempotent after a process dies halfway through?

## Summary

CLI PHP is a process contract: arguments, environment, working directory, configuration, standard streams, side effects, and exit status all matter. Keep input adaptation separate from business logic, make partial progress and retry behavior explicit, and measure the deployed invocation. The next chapter follows PHP across a web server boundary through CGI and FastCGI.

## References

- [PHP Manual: Command-line usage](https://www.php.net/manual/en/features.commandline.php)
- [PHP Manual: Command-line options](https://www.php.net/manual/en/features.commandline.options.php)
- [PHP Manual: CLI SAPI](https://www.php.net/manual/en/features.commandline.php)
- [PHP Manual: `exit`](https://www.php.net/manual/en/function.exit.php)
- [PHP Manual: Standard streams](https://www.php.net/manual/en/features.commandline.io-streams.php)
- [PHP Manual: Built-in web server](https://www.php.net/manual/en/features.commandline.webserver.php)
