---
book: The Complete Modern PHP Engineering Book
volume: 2
volume_title: PHP LANGUAGE FUNDAMENTALS
chapter: 19
title: Error Handling
slug: error-handling
status: complete
summary: ../../_ai/chapter-summaries/019-error-handling-summary.md
---

# Chapter 19 — Error Handling

## Why This Matters

Every useful PHP program encounters errors: invalid input, a missing file, a failed database connection, an exhausted disk, a timeout, a programming bug, or a process that is terminated halfway through a job. Error handling is the design of what the program communicates, what it records, what it can recover, and what it must refuse to pretend succeeded.

PHP's historical error system makes this especially important. Warnings, notices, fatal errors, exceptions, and engine Error objects have different origins and control-flow behavior. Treating every emitted message as harmless noise, or turning every message into the same exception without a policy, produces systems that are either silent or impossible to operate.

## Mental Model

Separate four questions:

1. How was the failure detected: return value, emitted error, Throwable, timeout, or external health signal?
2. Is the condition expected at this boundary, or is it a programming defect?
3. Can the operation be retried, corrected, compensated, or safely abandoned?
4. What should the caller see, and what diagnostic should operators receive?

The path should be deliberate:

~~~text
detect
  -> classify
  -> record useful context
  -> recover or stop
  -> translate at the boundary
~~~

Logging is not recovery. Displaying a friendly message is not diagnosis. A successful HTTP response is not proof that every side effect succeeded.

## Core Concept

PHP has configurable error levels, including notices, warnings, user-generated errors, and fatal categories. The exact set and behavior have changed across PHP versions. Modern PHP also throws Error and TypeError objects for many programming and type failures. All objects that can be thrown implement Throwable, but an emitted warning is not automatically a Throwable.

The most important production settings are usually:

~~~php
error_reporting(E_ALL);
ini_set('display_errors', '0');
ini_set('log_errors', '1');
~~~

The exact settings belong in deployment configuration, not arbitrary application files. Development should make failures visible. Production should avoid displaying stack traces or secrets to users while preserving structured diagnostics in a protected log or error-monitoring system. Confirm SAPI and configuration behavior in the environment you deploy.

## How It Works

### Return values versus emitted errors

An API may report failure as a sentinel:

~~~php
$contents = file_get_contents($path);
if ($contents === false) {
    // Inspect the boundary and decide whether this is expected.
}
~~~

Do not use truthiness when an empty string is valid. Some APIs emit a warning as well as returning false. If a warning is part of an expected branch, the design should make that behavior explicit and testable rather than allowing it to disappear into logs.

### Error handlers

set_error_handler() can route many non-fatal PHP errors to application code:

~~~php
set_error_handler(
    static function (
        int $severity,
        string $message,
        string $file,
        int $line,
    ): bool {
        if (!(error_reporting() & $severity)) {
            return false;
        }

        throw new ErrorException($message, 0, $severity, $file, $line);
    },
);
~~~

Returning false lets PHP's normal handler continue. Returning true says the handler dealt with that error. This mechanism does not make every failure catchable: parse errors and several engine or compile-time fatal categories occur before or outside the handler's useful scope. A shutdown function can inspect the last error for some terminal failures, but it cannot reliably resume a broken computation.

Converting warnings to ErrorException can be valuable inside a carefully bounded operation, such as a file import. It can also be harmful if a third-party library uses warnings as a documented probe or if the handler remains installed across unrelated code. Install handlers narrowly, restore them in finally, and define which severities become exceptions.

### Shutdown diagnostics

A shutdown function is useful for last-chance telemetry:

~~~php
register_shutdown_function(static function (): void {
    $last = error_get_last();

    if ($last !== null && in_array(
        $last['type'],
        [E_ERROR, E_PARSE, E_CORE_ERROR, E_COMPILE_ERROR],
        true,
    )) {
        error_log(json_encode([
            'event' => 'terminal_php_error',
            'type' => $last['type'],
            'message' => $last['message'],
            'file' => $last['file'],
            'line' => $last['line'],
        ], JSON_THROW_ON_ERROR));
    }
});
~~~

This is diagnostics, not a transaction. It cannot undo a committed database write or guarantee that a response can be changed. Avoid performing complex work in shutdown code; the process may be in a damaged state.

## What PHP Does

PHP evaluates the active error-reporting configuration and routes errors through its internal machinery. display_errors controls whether messages are sent to the output channel; log_errors controls whether they are written to the configured error log. Neither setting validates the program's behavior or guarantees that logs are safely handled.

The error-control operator can suppress output for an expression, but suppression is not a recovery strategy and can hide important operational signals. Prefer handling a documented return value or a narrowly scoped handler. Do not use suppression to make an insecure or undefined operation look reliable.

A user-defined error handler does not replace exception handling. A catch block handles thrown Throwable values; an error handler handles the categories it is registered to receive. At an application boundary, handle both according to their contracts.

## What Zend Does

The Zend Engine detects failures at different phases: parsing and compilation can fail before normal execution; execution can emit a warning or throw an Error; an extension can return a failure or raise an engine-level condition. This phase distinction explains why code cannot use a normal try/catch around every possible syntax or startup failure.

The engine also carries a stack and current execution context used to build traces. A trace is valuable for diagnosis but can contain file paths, arguments, and implementation details. Treat it as restricted diagnostic data, not as a client response.

## Minimal Example

This importer turns a warning-prone file read into a small, explicit result at one boundary:

~~~php
<?php

declare(strict_types=1);

/** @return array{ok: true, contents: string}|array{ok: false, reason: string} */
function readImport(string $path): array
{
    set_error_handler(
        static function (int $severity, string $message, string $file, int $line): never {
            throw new ErrorException($message, 0, $severity, $file, $line);
        },
        E_WARNING | E_NOTICE,
    );

    try {
        $contents = file_get_contents($path);

        if ($contents === false) {
            return ['ok' => false, 'reason' => 'read_failed'];
        }

        return ['ok' => true, 'contents' => $contents];
    } catch (ErrorException) {
        return ['ok' => false, 'reason' => 'read_failed'];
    } finally {
        restore_error_handler();
    }
}
~~~

The public result is safe to use in a batch workflow, while the internal diagnostic can be logged at the boundary. In a real importer, validate the path against an allowed directory, enforce a size limit, and distinguish “not found” from “permission denied” if the caller can act differently.

## Practical Example: Translating Failures

A command-line import may classify failures differently:

~~~php
$result = readImport($path);

if (!$result['ok']) {
    fwrite(STDERR, "Import failed: {$result['reason']}\n");
    exit(2);
}

fwrite(STDOUT, "Read " . strlen($result['contents']) . " bytes\n");
~~~

A web adapter might return a generic 400 or 503 response instead. The importer should not know about HTML, HTTP status codes, or terminal colors. Translation belongs at the outer boundary, where transport and security policy are known.

## Production Example: A Safe Top-Level Boundary

An outer boundary often catches unhandled Throwable values so the process can report a controlled failure:

~~~php
try {
    $response = application()->handle($request);
    sendResponse($response);
} catch (Throwable $throwable) {
    reportThrowable($throwable); // redact and correlate
    sendInternalFailureResponse();
}
~~~

This catch-all is a last boundary, not an invitation to continue as if nothing happened. It should preserve the process's failure signal, avoid leaking details, and ensure that a worker or request lifecycle is in a known state. A long-running worker may need to discard request-local state or terminate after a fatal class of failure. A queue consumer should acknowledge a message only after durable success.

## Bad Example

~~~php
<?php

error_reporting(0);

$config = parse_ini_file('/etc/app/config.ini');
$dsn = $config['dsn'];
$pdo = new PDO($dsn);

echo 'Application started';
~~~

This hides configuration errors, assumes parsing succeeded, dereferences a possibly false value, and announces success before the dependency is usable. The first visible symptom may be an unrelated failure much later. Suppression changes observability, not correctness.

## Better Example

~~~php
<?php

declare(strict_types=1);

function loadDsn(string $path): string
{
    $config = parse_ini_file($path, true, INI_SCANNER_TYPED);

    if ($config === false || !isset($config['database']['dsn'])) {
        throw new RuntimeException('Database configuration is invalid');
    }

    $dsn = $config['database']['dsn'];
    if (!is_string($dsn) || $dsn === '') {
        throw new RuntimeException('Database DSN is invalid');
    }

    return $dsn;
}
~~~

The caller can catch this at a startup boundary, log a correlation ID and stop the process. The message is intentionally useful to operators without including credentials. Configuration validation should happen before accepting traffic.

## Edge Cases

- A warning can be emitted before a function returns false. Test and observe both channels.
- A handler cannot catch every error category, especially failures during parsing or compilation.
- An error handler that throws can change a library's documented control flow. Scope and restore it.
- A shutdown function runs late; it cannot roll back effects or guarantee output.
- display_errors must not expose paths, SQL, stack traces, or secrets to an untrusted client.
- Logging can itself fail because disk, permissions, or a remote collector is unavailable. Keep a minimal fallback and do not recursively log logging failures.
- An error message may contain user input. Use structured fields, size limits, and redaction rather than concatenating arbitrary content.
- A warning caused by a timeout or unavailable service may be retryable, but a malformed configuration is usually not.

## Performance

The normal path should not install handlers, build large traces, or format expensive diagnostic payloads unnecessarily. Error paths are usually less performance-sensitive than successful paths, but a malformed request can deliberately exercise them at high volume. Apply request and payload limits before expensive work, and rate-limit repeated identical reports.

Capturing stack traces, serializing large arguments, and synchronously sending telemetry can add latency to an already failing request. Prefer compact structured events with correlation identifiers. Measure logging overhead and queue or batch telemetry when the failure volume can be large.

## Security

Keep display_errors disabled for production responses and protect logs with access controls and retention rules. Redact passwords, tokens, cookies, authorization headers, personal data, and raw request bodies unless there is a documented need and protection. Do not use a user-provided message as a log format string, and do not let an attacker inject newlines or unbounded fields into text logs without structured encoding.

Error handling must fail closed for authorization and payment decisions. If policy data cannot be loaded, “unknown” is not “allowed.” Do not return different detailed errors when that would reveal whether a sensitive account or resource exists. Use generic client messages and detailed, access-controlled operator diagnostics.

## Database Interaction

A database error has several possible meanings: invalid SQL, a constraint violation, a deadlock, a network failure, or a transaction that has become unusable. Classify it at the database adapter and preserve the original cause for diagnostics. Do not catch a database failure, log it, and continue issuing statements on a transaction whose state is unknown.

Error handling cannot create a transaction. Use the database's transaction API for atomic work, roll back on failure, and define what happens to external effects. If a commit result is ambiguous because the connection died, an idempotency key or reconciliation process may be needed before retrying.

## Concurrency

Two workers can report the same error at once, and retries can turn a transient error into a load storm. Include a correlation or operation identifier, classify retryability, and use bounded backoff outside this chapter's low-level handler. Do not use a process-local “already logged” array as a distributed deduplication mechanism.

In a long-running worker, clear request-local context after each job and decide which failures terminate the worker. An exception or error from one job must not leak authentication, logging context, or partial mutable state into the next job.

## Testing

Test each reporting mechanism deliberately:

~~~php
public function testMissingImportReturnsAStableReason(): void
{
    self::assertSame(
        ['ok' => false, 'reason' => 'read_failed'],
        readImport('/does/not/exist'),
    );
}
~~~

Use a temporary file for successful reads. Add tests for empty files, permission failures where the platform permits them, malformed configuration, warning-to-exception conversion, restoration of the previous error handler, and top-level translation. Capture logs through a test sink and assert that secrets and raw request data are absent. Integration tests should verify the actual production-like PHP error configuration.

## Common Mistakes

- Setting error_reporting(0) to remove noise.
- Confusing display_errors with handling or recovery.
- Catching only Exception and missing Error or TypeError at a true top-level boundary.
- Installing a global error handler that changes third-party behavior.
- Logging a full exception, request, or configuration object without redaction.
- Returning success after a warning or failed dependency call.
- Retrying every error without classifying idempotency and retryability.
- Treating shutdown diagnostics as a rollback mechanism.

## Senior Engineer Thinking

An error policy should be visible in the interface: return a result when failure is an expected branch, throw when the caller cannot sensibly continue, and translate only at a boundary that knows the caller's protocol. Every failure path should answer what was persisted, what can be retried, what operators can see, and what the client is allowed to learn.

Good error handling reduces ambiguity. It preserves cause, adds context, avoids duplicate noise, and leaves the system in a known state. It never converts “we do not know” into “success.”

## Exercises

1. Wrap a warning-producing file API in a narrow adapter that returns a typed result. Restore the prior handler in all paths.
2. Design a structured error event for a failed payment without logging card data, credentials, or the full request body.
3. Build a top-level CLI boundary that maps validation, dependency, and unexpected failures to distinct exit codes.
4. Test a long-running worker for context leakage after one job fails.

## Review Questions

1. How do display_errors and log_errors differ?
2. Why can a normal error handler not catch every parse or fatal error?
3. When is converting an emitted warning to ErrorException useful, and when can it be harmful?
4. Why is logging not recovery?
5. What should happen when authorization data cannot be loaded?
6. Why can a retry after an ambiguous database commit create a duplicate side effect?
7. What additional error-handling concern exists in a long-running worker?

## Summary

Error handling is a policy for detection, classification, diagnosis, recovery, and translation. PHP's emitted errors, return values, Error objects, and exceptions are different mechanisms. Keep handlers narrow, configure production visibility safely, preserve causes, redact diagnostics, and treat shutdown hooks as last-chance telemetry only. Database transactions, idempotency, retry policy, and worker lifecycle still need explicit designs beyond the error channel.
