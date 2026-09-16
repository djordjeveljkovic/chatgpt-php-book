---
book: The Complete Modern PHP Engineering Book
volume: 1
volume_title: THE PHP MENTAL MODEL
chapter: 4
title: How PHP Applications Execute
slug: how-php-applications-execute
status: complete
summary: ../../_ai/chapter-summaries/004-how-php-applications-execute-summary.md
---

# Chapter 4 — How PHP Applications Execute

## Why This Matters

When a PHP application fails, the visible symptom is often far away from the cause. A browser may show a blank page, a command may return a non-zero exit code, or a queue may report that a job timed out. Those symptoms do not tell us whether the problem occurred while source was being loaded, while application code was running, while an external service was being called, or after the application had already produced its result.

The phrase “the application ran” also hides several different events. Some component selected an entry point. A PHP runtime was started or reused. Source files were loaded and made executable. Input was turned into application values. Code performed work and produced output or side effects. Finally, control returned to the surrounding system—or did not, because the process crashed, was terminated, or waited indefinitely.

An experienced engineer can reconstruct that sequence. That makes debugging and design more precise: the same domain operation can be used from HTTP, CLI, a queue, or a scheduled task, while each boundary handles its own input, output, lifetime, and failure policy.

## Mental Model

Think of execution as a pipeline with two dimensions:

```text
Source and configuration
        ↓
Entry point selects a program
        ↓
PHP loads, parses, and prepares source
        ↓
Runtime executes the prepared program
        ↓
Application reads input and performs work
        ↓
Output and side effects cross process boundaries
        ↓
Request, job, or process ends
```

The first dimension is the source-to-execution path. It explains how text in a `.php` file becomes instructions the runtime can execute. The second is the application lifecycle. It explains what surrounds that execution: startup, bootstrap, input handling, application work, response or acknowledgment, cleanup, and termination.

These are related but not identical. A source file can be prepared successfully and still fail during execution. An application can finish its domain work but fail while sending a response. A long-running worker can execute many jobs in one operating-system process, while a CLI command may execute once and exit. The lifecycle determines which state and resources survive from one unit of work to the next.

The most useful boundary is this:

> PHP source describes behavior; an entry point and a process provide the conditions under which that behavior occurs.

## Core Concept

### From source text to executable work

At a conceptual level, PHP execution has four stages:

1. **Load** — the selected entry script and any required files are found and read.
2. **Parse and prepare** — the runtime checks the source structure and prepares an executable representation.
3. **Execute** — the runtime evaluates statements, calls functions, creates values, and handles control flow.
4. **Expose effects** — bytes, exit status, files, database changes, messages, and other effects become visible outside the running code.

“Prepare” is deliberately broad. PHP has a parser, compiler, executable instructions, and optional opcode caching, but the implementation details belong in Volume IV. The important mental model here is that the runtime does not treat source text as a sequence of instructions that the operating system directly understands. It must first understand the source and create an internal form that can be executed.

If the source is syntactically invalid, execution does not reach the application statements that follow the invalid part. If preparation succeeds but a function later receives an invalid value, the failure occurs during execution. If execution succeeds but a database rejects a write, the failure occurs at an external boundary. These are different failure classes and require different evidence.

OPcache can change the cost of the preparation stages. The PHP manual describes it as storing precompiled script bytecode in shared memory so PHP does not need to load and parse every script on every request. That optimization changes how work is reused; it does not change the application’s source-level contract or make external side effects safe to repeat.

### Loading is part of application behavior

A real application rarely consists of one isolated file. The entry script loads configuration, an autoloader, framework or library code, and application modules. A missing file, an incorrect path, an unavailable extension, or a configuration error can prevent the intended application code from running.

Composer’s autoloader is a useful example of this boundary. It does not make classes exist magically. The entry point includes the generated autoloader; later class references cause the loader to map names to files or other registered loading rules. The details belong to the dependency-management layer, but the lifecycle lesson is general: bootstrap code establishes the conditions required by the rest of the program.

### Execution produces more than output

Output is only one category of effect. A PHP program may:

- write an HTTP response or CLI text;
- return an exit status to its caller;
- insert or update database records;
- publish a queue message;
- write a file or log entry;
- call another service;
- mutate process-local state;
- deliberately produce no direct output.

The runtime can report that a script completed while an external effect still failed, or an effect can succeed while the process later fails during cleanup. Reliable applications therefore define what “completed” means for each entry point. For a command, it may mean that all records were imported and the exit status is zero. For a queue job, it may mean that the message’s work is durable and the message can be acknowledged. For HTTP, it may mean that the response has been committed, even though work after that point has a different reliability profile.

## How It Works

### 1. An entry point is selected

An entry point is the boundary through which an external actor starts a particular application flow. Common examples are:

```text
HTTP request       → public/index.php
CLI command        → bin/console or a script
Queue delivery     → worker loop and job handler
Scheduled task     → cron/systemd/container invocation
Test               → test runner and test case
```

The file name is a convention, not the definition. What matters is that some launcher chooses code and supplies an execution context. PHP CLI can execute a file, receive code with `-r`, or read code from standard input; in each case the CLI SAPI is the entry mechanism. A web server commonly passes requests to a PHP integration such as PHP-FPM, which manages PHP worker processes and accepts requests through the FastCGI boundary.

The entry point is responsible for translation. An HTTP entry point translates a request into values such as a route, authenticated identity, headers, and body. A CLI entry point translates arguments and environment variables. A queue entry point translates a message into a job payload. The application core should receive validated domain inputs rather than depend on every transport’s representation.

### 2. The process receives an environment

The operating system starts a process, or an existing managed process receives a unit of work. The process has an executable, arguments, environment variables, a current working directory, file descriptors, permissions, and resource limits. PHP configuration and enabled extensions further shape what the runtime can do.

This explains why “it works locally” is weak evidence. The local command may use a different PHP binary, `php.ini`, working directory, extension set, filesystem permissions, timezone, or environment variables. The source is only one input to execution.

The process also has boundaries. PHP variables live inside the process. A database connection, HTTP call, file write, or queue acknowledgment crosses into another system. Data must be serialized, transmitted, accepted, and possibly committed there. A successful function call in PHP is not the same thing as a durable result in the external system.

### 3. Bootstrap establishes application state

Most entry points perform some form of bootstrap:

```text
read configuration
    ↓
load dependencies and classes
    ↓
create clients, containers, or registries
    ↓
register error and shutdown behavior
    ↓
dispatch the current unit of work
```

Bootstrap should establish infrastructure, not secretly perform the business operation. A web request may construct a router and a database client. A command may construct an importer. A worker may initialize logging and a queue connection. Keeping the operation separate makes it possible to invoke the same business rule from multiple entry points and to test it without starting the entire environment.

### 4. Input is translated and validated

The application then obtains input from its boundary. Input is not automatically trustworthy merely because PHP can represent it as a string, array, or object. The boundary should establish the expected shape, types, authorization context, and limits before application code performs consequential work.

Validation and domain rules are related but different. “The field is an integer” is a boundary concern. “The requested quantity does not exceed available stock” is a domain rule. Both matter, but placing all checks in a controller makes the rule difficult to reuse from a CLI or worker.

### 5. Application code performs work

After translation, application code coordinates domain logic and infrastructure. It may read from a database, call a payment provider, publish an event, or calculate a result. At this stage, ordinary PHP semantics—function calls, objects, exceptions, references, and values—are active inside the process.

The runtime does not know that a sequence of calls constitutes “placing an order.” It executes the program’s instructions. The application and its dependencies define the larger operation, including which steps must be atomic, which failures are retryable, and what must be recorded for recovery.

### 6. Results cross the boundary

The final result is adapted to the caller:

- an HTTP flow creates a status, headers, and body;
- a CLI flow writes human- or machine-readable output and returns an exit code;
- a worker acknowledges, retries, rejects, or dead-letters a message;
- a scheduled task records success or failure for its scheduler and operator;
- a test runner records assertions and process status.

This translation is why a reusable application service should not print an HTTP response or call `exit()` merely because one caller is a web request. Those actions belong to the outer entry-point adapter unless the service’s explicit contract requires them.

### 7. Cleanup and termination occur

At the end of a short-lived invocation, PHP releases request-local resources as the process or request context ends. Connections may be closed, temporary files removed, buffers flushed, and shutdown handlers invoked according to the runtime and surrounding system.

The exact cleanup behavior is not a substitute for explicit resource management. If a database transaction must be committed or rolled back, the application should do that deliberately. If a lock must be released, the owning system’s contract should be followed. “The process will eventually end” is not a safe recovery strategy for a worker that may remain alive for hours.

## Entry Points and Process Boundaries

### HTTP entry points

An HTTP request normally travels through several components:

```text
client
  ↓
reverse proxy / web server
  ↓  FastCGI or another integration
PHP process manager and worker
  ↓
front controller
  ↓
router, middleware, application code
  ↓
response
```

The front controller is an application entry point, not the web server itself and not PHP-FPM. The proxy may terminate TLS and enforce request limits. The process manager may choose a worker. The PHP application interprets the request and creates a response. A failure in one layer can prevent later layers from running.

An important consequence is that a request is a unit of work, not necessarily an operating-system process. In a managed web deployment, a pool of workers may handle requests over time. Request-local variables should not be treated as shared application state, but a worker process can still retain resources or static state longer than one request. The safe design depends on the actual lifecycle and deployment model.

### CLI entry points

CLI execution is often easier to observe because the shell starts the PHP executable directly:

```text
shell / scheduler
        ↓
PHP CLI executable
        ↓
script entry point
        ↓
application work
        ↓
stdout, stderr, exit status
```

The command receives arguments and environment variables, and its output is consumed by a terminal, a pipe, a log collector, or a scheduler. A command can be invoked without any web server. This makes CLI a separate execution context, not merely “web PHP without a browser.”

### Queue and worker entry points

A worker usually has a different shape:

```text
worker process starts
        ↓
bootstrap once
        ↓
receive message
        ↓
translate and execute job
        ↓
acknowledge or retry
        ↺ receive another message
```

The loop creates a longer process lifetime. Objects, static properties, caches, open resources, and accidental references may survive between jobs. A worker must define what is reset per message and what is intentionally shared. It also must treat duplicate delivery, termination during a job, and acknowledgment timing as part of correctness.

### Scheduled and test entry points

A scheduler may launch a new process for every run, or it may invoke a long-lived service. A test runner may load the application repeatedly in one process or isolate tests according to its own model. The same PHP class can therefore experience different lifetimes depending on the launcher.

Tests should make that context explicit. A unit test can call a pure application component directly. An integration test can cross a database or filesystem boundary. An end-to-end test can exercise the real entry point. Each level answers a different question.

## Minimal Example

The following application service knows nothing about HTTP or the shell:

```php
<?php

declare(strict_types=1);

final class Greeting
{
    public function message(string $name): string
    {
        $name = trim($name);

        if ($name === '') {
            throw new InvalidArgumentException('A name is required.');
        }

        return "Hello, {$name}!";
    }
}
```

One CLI entry point can adapt its boundary:

```php
<?php

declare(strict_types=1);

require __DIR__ . '/../vendor/autoload.php';

$name = $argv[1] ?? '';

try {
    echo (new Greeting())->message($name), PHP_EOL;
    exit(0);
} catch (InvalidArgumentException $exception) {
    fwrite(STDERR, $exception->getMessage() . PHP_EOL);
    exit(2);
}
```

An HTTP adapter could call the same `Greeting::message()` method and translate its result into a response. The service does not need to know whether the caller’s input came from `$argv`, JSON, or a test fixture. The adapters own transport details; the service owns the operation.

The example also shows that an exception is not itself a complete user-facing result. The CLI adapter chooses an error stream and exit code. Another adapter could choose an HTTP status and JSON body. The boundary gives the failure its context.

## Production Example

Imagine an order endpoint that accepts `POST /orders`. A production execution might look like this:

```text
1. Proxy accepts the connection and forwards the request.
2. A PHP worker receives the request through the web integration.
3. The front controller loads configuration and dependencies.
4. Routing and middleware identify the operation and user.
5. The request body is decoded and validated.
6. Application code reads inventory and writes the order.
7. The database commits, or the operation fails and is rolled back.
8. The application creates an HTTP response.
9. The worker returns to its managed state for another request.
```

Each line has a distinct boundary and timeout. A proxy timeout does not prove that the database rolled back. A client retry does not prove that the first request never committed. A PHP exception does not automatically undo an already committed external side effect. The system needs an explicit policy for idempotency, transactions, retries, and observability.

The same order operation might be started by a CLI reconciliation command or a queue message. Reusing the domain operation is valuable, but reusing the entire HTTP lifecycle is not. The command and worker need their own input, output, timeout, and retry behavior.

## Bad Example and Better Example

A boundary leak often looks harmless:

```php
final class ImportUsers
{
    public function run(array $rows): void
    {
        foreach ($rows as $row) {
            // Persist the user...
            echo "Imported {$row['email']}\n";
        }
    }
}
```

This class has coupled the operation to one output channel. A web caller may receive diagnostic text mixed into its response. A queue worker may flood its logs. A test must capture output merely to verify persistence.

A better boundary returns facts and lets the caller decide how to report them:

```php
final class ImportUsers
{
    /** @return list<string> Imported email addresses. */
    public function run(array $rows): array
    {
        $imported = [];

        foreach ($rows as $row) {
            // Persist the user...
            $imported[] = $row['email'];
        }

        return $imported;
    }
}
```

A CLI adapter can print a count, a web adapter can return JSON, and a worker can record structured metrics. In a very large import, returning every email would create unnecessary memory pressure; the same design could instead return a count or emit progress through an explicit collaborator. The boundary should match the workload.

## Edge Cases

Several events are easy to confuse:

- **Parse or preparation failure:** the selected source cannot be prepared, so the intended statements do not run.
- **Runtime exception or error:** execution began, but a later operation failed or was rejected.
- **Process termination:** the operating system, supervisor, timeout, or operator stopped the process; application cleanup may be incomplete.
- **External timeout:** PHP stopped waiting for a dependency, but the dependency may still have completed its work.
- **Partial success:** one side effect succeeded before a later side effect failed.
- **Duplicate execution:** a client, scheduler, or queue may start the same logical operation again.

The remedy is not one universal retry. First identify the stage and boundary. Then decide whether the operation is safe to repeat, whether a transaction or idempotency key is needed, and which durable evidence should be recorded.

## Performance

Execution cost has both startup and work components:

```text
total cost ≈ bootstrap cost + application work + boundary cost
```

For a short CLI command, bootstrap may be a noticeable fraction of total time. For a request that performs a large database scan, bootstrap may be negligible beside the query. For a worker, bootstrap can be amortized across many jobs, but retained memory and stale resources become operational concerns.

OPcache can reduce repeated source preparation, while autoloading and bootstrap still have costs. The right optimization depends on measurements: request latency, command duration, worker throughput, memory over time, database timings, and external-service latency. Do not optimize the parser when the dominant cost is a remote call.

## Security

Every entry point has a different trust boundary:

- HTTP input is controlled by clients and intermediaries.
- CLI arguments may be supplied by operators, schedulers, or untrusted file names.
- Queue payloads may be malformed, stale, duplicated, or produced by a compromised publisher.
- Environment variables and configuration may contain secrets and may be misconfigured.
- A FastCGI listener must be protected; the PHP manual warns that an untrusted client able to open an FPM connection can control request configuration and execute arbitrary code.

Validate at the boundary, use least privilege for the process, protect runtime control interfaces, and avoid leaking source paths or secrets in error output. Process separation limits accidental state sharing, but it is not a replacement for authentication, authorization, or operating-system security.

## Testing

Test the pipeline at the level needed by the risk:

- unit-test application rules without a transport;
- integration-test database, filesystem, and message boundaries;
- test CLI adapters for argument parsing and exit codes;
- test HTTP adapters for request translation and response mapping;
- test worker behavior for retry, acknowledgment, duplicate delivery, and shutdown;
- run a smoke test through the deployed entry point to catch configuration and bootstrap errors.

A passing unit test proves that a component behaves under its test inputs. It does not prove that the deployed PHP binary loads the expected extension, that the process can reach the database, or that the proxy forwards the request correctly. Those require tests at the corresponding boundary.

## Common Mistakes

- Treating PHP source as if the operating system executes it directly.
- Assuming every web request creates a new process.
- Assuming every PHP process ends after one unit of work.
- Putting HTTP, CLI, or queue behavior inside reusable domain code.
- Treating a successful function return as proof that a remote side effect is durable.
- Retrying an operation without checking idempotency and partial success.
- Comparing local and production behavior without comparing binaries, configuration, permissions, and environment.
- Using process termination as ordinary application control flow.
- Testing only the application core and never testing the entry-point translation.

## Senior Engineer Thinking

When debugging an execution problem, draw the path before changing code:

```text
Who started it?
What process ran it?
Which entry point was selected?
What source and configuration were loaded?
What input crossed the boundary?
Which operation was attempted?
Which external system was contacted?
What result crossed back?
What survived after the unit of work ended?
```

This sequence turns vague reports into testable hypotheses. “The order endpoint timed out” becomes a set of questions: Did the request reach the proxy? Did PHP begin execution? Was bootstrap slow? Did the database commit before the timeout? Did a retry create a duplicate? Did the worker remain healthy afterward?

The same reasoning helps with architecture. Keep transport-specific concerns at the edge, make important application operations explicit, and document which state is request-local, process-local, or durable in an external system. A process boundary is a fact about lifetime and communication—not a decorative box in a diagram.

## Exercises

1. Draw the execution path for a CLI command that imports a CSV file. Mark where the shell, PHP process, filesystem, database, and exit status are involved.
2. Take one HTTP controller from an application and identify which lines translate input, which lines perform domain work, and which lines translate output. What could be reused by a queue consumer?
3. List three failures that could occur after an external payment provider accepts a charge but before the HTTP response reaches the client. For each, decide what evidence and retry policy would be required.

## Review Questions

1. What is the difference between loading source, preparing it, executing it, and exposing its effects?
2. Why is an entry point an adapter rather than the whole application?
3. Why does a web request not necessarily correspond to a new operating-system process?
4. What changes when a worker handles many jobs in one process?
5. Why should reusable application code avoid printing transport-specific output or calling `exit()`?
6. Why does a successful PHP function call not prove that a database or remote service committed the intended effect?
7. Which parts of an execution failure can be tested with a unit test, and which require an integration or end-to-end test?

## Summary

A PHP application executes through a pipeline: an entry point selects a program, a process supplies its environment, the runtime loads and prepares source, application code executes, effects cross process boundaries, and the result is translated for the caller.

The lifecycle differs by entry point. HTTP, CLI, queue, scheduled, and test flows have different input and output contracts. A request is a unit of work, not automatically a process; a worker process may handle many units of work. That distinction determines what state, resources, and failures can survive.

The practical mental model is to separate reusable application work from boundary adapters, then investigate failures by stage and process boundary. The next chapter can make the CLI-versus-web contrast concrete; the later runtime volume can explain parsing, compilation, opcodes, PHP-FPM, and request internals in depth.

## Sources and Further Reading

- [PHP Manual: Command line usage](https://www.php.net/manual/en/features.commandline.usage.php)
- [PHP Manual: FastCGI Process Manager (FPM)](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: OPcache](https://www.php.net/manual/en/book.opcache.php)
- [PHP Manual: Runtime Configuration — OPcache](https://www.php.net/manual/en/opcache.configuration.php)
