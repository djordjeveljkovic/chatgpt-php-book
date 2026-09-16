---
book: The Complete Modern PHP Engineering Book
volume: 1
volume_title: THE PHP MENTAL MODEL
chapter: 5
title: CLI vs Web PHP
slug: cli-vs-web-php
status: complete
summary: ../../_ai/chapter-summaries/005-cli-vs-web-php-summary.md
---

# Chapter 5 — CLI vs Web PHP

## Why This Matters

The same PHP language can run a command from a terminal, handle an HTTP request through PHP-FPM, or process messages inside a long-lived worker. The source code may look similar in all three cases, but the execution contract is different.

That contract determines where input comes from, where output goes, how configuration is selected, how long memory remains alive, what a failure means, and which test gives us confidence. A function that reads `$_POST`, prints an HTML fragment, and assumes the process will end immediately is coupled to one environment. It is not automatically reusable in a command, a queue consumer, or a test.

The practical lesson is not that one SAPI is better than another. It is that an experienced PHP engineer identifies the execution context before designing the boundary around the application logic.

## Mental Model

Start with the same application logic surrounded by different adapters:

```text
                    ┌───────────────┐
Terminal ──────────▶│ CLI adapter   │──┐
                    └───────────────┘  │
                                         ▼
                    ┌───────────────┐  ┌──────────────────┐
HTTP server ───────▶│ Web adapter   │─▶│ Application logic │
                    └───────────────┘  └──────────────────┘
                                         ▲
                    ┌───────────────┐  │
Queue supervisor ──▶│ Worker adapter│──┘
                    └───────────────┘
```

The adapter translates an environment-specific event into explicit application inputs. It also translates the result into an environment-specific outcome:

| Context | Input | Output | Typical lifetime | Failure signal |
| --- | --- | --- | --- | --- |
| CLI command | Arguments, environment, standard input, files | Standard output/error, exit status, files or side effects | One invocation | Exit code and diagnostics |
| Web request through FPM | HTTP method, URL, headers, body, uploaded files | Status, headers, response body, side effects | Usually one request at a time per FPM child | HTTP response, logs, process failure, or upstream timeout |
| Application worker | Queue message, broker metadata, signals, environment | Acknowledgment, retry, rejection, logs, side effects | Many messages in one process | Ack/nack, retry policy, visibility timeout, or process exit |

These are not three different languages. They are three different ways of starting and supervising the PHP runtime.

The phrase “web PHP” also needs precision. A browser normally talks to a web server or reverse proxy. That server forwards a request to a PHP entry point, commonly PHP-FPM over FastCGI. PHP-FPM manages pools of child processes; those children execute application code for requests. An FPM child is a server worker, but it is not the same thing as an application-level queue worker that intentionally remains alive while consuming messages.

## Core Concept

### SAPI is the boundary around the runtime

PHP uses a Server Application Programming Interface, or SAPI, to connect the core runtime to its host environment. The CLI SAPI is designed for shell applications. FPM is a FastCGI implementation with process-management features for serving web requests. Other SAPIs exist, but the important distinction here is the host contract.

The SAPI influences details such as:

- which input channels are available;
- which output streams are meaningful;
- how configuration is loaded;
- how requests or invocations begin and end;
- what the process supervisor considers success or failure.

The PHP language does not require a browser. `echo` writes output, but the meaning of those bytes depends on the SAPI and the process around it. In CLI they normally go to standard output. In a web request they become response bytes, subject to headers, buffering, the web server, and the HTTP protocol.

### A process boundary is a design boundary

For a short command, the usual mental model is:

```text
start process → load configuration → execute script → flush output → exit
```

For a traditional FPM request, a useful conceptual model is:

```text
FPM child waits → receives request → initializes request state
               → runs application → sends response → cleans request state
               → waits for another request or is recycled
```

The child process may serve many requests over its lifetime. Therefore, “the request ends” and “the process ends” are different events. Request-local variables normally disappear as request execution is torn down, but process-level resources, extension state, files, connections, static variables, and memory retained by application code can outlive one request.

For a long-running application worker, the model is different again:

```text
start process → boot application → receive message
             → handle message → acknowledge or retry
             → repeat until shutdown or failure
```

In this model, every iteration is a new logical job inside the same operating-system process. State that is accidentally retained can affect later jobs. A worker must also handle signals, graceful shutdown, memory growth, and repeated delivery deliberately.

## How It Works

### CLI execution

The CLI executable can run a file, evaluate code passed with `-r`, or read code from standard input. A command can receive arguments in `$argv` and `$argc`, read bytes from `STDIN`, and write to `STDOUT` or `STDERR`.

```php
<?php

declare(strict_types=1);

if ($argc !== 2) {
    fwrite(STDERR, "Usage: php greet.php <name>\n");
    exit(64);
}

$name = trim($argv[1]);

if ($name === '') {
    fwrite(STDERR, "Name must not be empty.\n");
    exit(65);
}

fwrite(STDOUT, "Hello, {$name}!\n");
```

The command has an explicit success path and explicit failure paths. A shell, scheduler, or CI system can inspect the exit status. The message for a human can go to standard error while normal output remains machine-readable on standard output.

The PHP manual's [CLI usage documentation](https://www.php.net/manual/en/features.commandline.usage.php) describes these input modes and the CLI-specific behavior. One detail worth remembering is that command-line arguments are first parsed by PHP itself. A `--` separator is needed when an argument beginning with `-` should be passed to the script rather than interpreted as a PHP option.

### Web execution through FPM

A web request normally enters through an HTTP server. The server terminates or forwards the HTTP connection, applies web-server rules, and passes the request to PHP-FPM. FPM selects an available child in a pool, and that child runs the configured PHP entry point.

The application may access request data through superglobals such as `$_SERVER`, `$_GET`, `$_POST`, `$_COOKIE`, and `$_FILES`, or through a framework request object. Raw request bytes can be read from `php://input`. The response is made from a status, headers, and body, whether those are produced directly by PHP or by a framework response abstraction.

```php
<?php

declare(strict_types=1);

$name = trim((string) ($_GET['name'] ?? ''));

if ($name === '') {
    http_response_code(400);
    header('Content-Type: text/plain; charset=utf-8');
    echo "Missing name\n";
    return;
}

header('Content-Type: text/plain; charset=utf-8');
echo "Hello, {$name}!\n";
```

This example has a different output contract from the CLI command. `400` is useful to an HTTP client; it is not a replacement for a shell exit code. In production code, input parsing, validation, escaping, authentication, and response formatting should usually be delegated to well-defined application or framework boundaries rather than kept in a script like this.

PHP-FPM pools can use static, dynamic, or ondemand child-process management. The pool's `pm.max_children` limits the number of simultaneous requests served by that pool. This means an application can be perfectly correct in a single request and still experience queueing or upstream timeouts when all children are busy. Capacity is a property of the whole path—proxy, web server, FPM pool, database, and other dependencies—not just of PHP code.

The [PHP-FPM manual](https://www.php.net/manual/en/install.fpm.php) and [FPM configuration reference](https://www.php.net/manual/en/install.fpm.configuration.php) describe this process model and its pool controls.

### Application workers

An application worker is usually started by a process supervisor or a queue system. It may use a library or framework to receive messages, but the underlying PHP process is still running a loop.

```php
<?php

declare(strict_types=1);

while ($message = receiveMessage()) {
    try {
        handleMessage($message);
        acknowledge($message);
    } catch (Throwable $exception) {
        reportFailure($message, $exception);
        retryOrReject($message, $exception);
    }
}
```

This pseudocode hides the hard parts: what “receive” does when the broker is unavailable, when a message becomes visible again, whether acknowledgment happens before or after side effects, and how duplicate delivery is handled. A worker cannot assume that a caught exception means the message was safely processed or that the process is still healthy.

It also cannot assume that request cleanup will protect it. The worker should make per-message state local to the loop, release or reset resources where appropriate, and be restarted according to an intentional policy. Recycling after a bounded number of jobs or after a memory threshold can be a practical containment measure, but it does not replace fixing unbounded retention.

## Input and Output

### Input is not a global application API

Superglobals are convenient at a web boundary, and `$argv` is convenient at a CLI boundary. They become a problem when business logic reaches for them directly.

This function is difficult to reuse and test:

```php
function createUser(): void
{
    $email = $_POST['email'];
    // Validate and persist the user.
}
```

It assumes an HTTP POST, a particular field name, and a global mutable source. A CLI command cannot call it naturally. A unit test must manipulate global state. A queue message would need to imitate an HTTP request for no domain reason.

Prefer converting environment-specific input at the edge:

```php
final readonly class CreateUserInput
{
    public function __construct(public string $email)
    {
    }
}

function createUser(CreateUserInput $input): void
{
    // Validate the value according to the application's rules and persist it.
}
```

The web adapter can build `CreateUserInput` from a request. The CLI adapter can build it from arguments or a file. A worker can build it from a message. The shared function now has an explicit contract.

### Output has a consumer

Output is also an interface. Decide whether a result is intended for:

- a human reading a terminal;
- another program consuming stdout;
- an HTTP client expecting a status and a representation;
- a queue broker expecting acknowledgment;
- a log or metric system;
- no immediate consumer because the operation only creates a side effect.

Mixing these channels causes operational confusion. A CLI command that writes progress bars to stdout may corrupt a CSV pipeline. A web handler that prints debugging text before headers can produce a malformed response. A worker that treats a log line as acknowledgment has not actually told the broker the message succeeded.

Keep normal data, diagnostics, protocol output, and side effects conceptually separate. The exact mechanism varies, but the separation makes each adapter observable and testable.

## Configuration

Configuration often differs between `php` on a shell and the PHP runtime serving web traffic. They may use different `php.ini` files, enabled extensions, environment variables, working directories, users, and limits. The command `php --ini`, `php -i`, and a small diagnostic script can reveal the CLI configuration; a web diagnostic endpoint or FPM status/configuration inspection is needed to inspect the web path safely.

Do not assume that “PHP version” identifies the complete runtime. Two invocations can use the same version number while differing in SAPI, loaded extensions, configuration, user permissions, and current working directory. At a boundary, record the facts that matter:

```php
<?php

declare(strict_types=1);

fwrite(STDOUT, sprintf(
    "sapi=%s version=%s ini=%s\n",
    PHP_SAPI,
    PHP_VERSION,
    php_ini_loaded_file() ?: '(none)',
));
```

`PHP_SAPI` identifies the current SAPI, while `php_ini_loaded_file()` reports the loaded `php.ini` path when one is loaded. These are diagnostic facts, not configuration strategy.

Some directives are set only at system or per-directory levels; others may be changed during script execution. `ini_set()` changes a permitted setting for the current script execution and its value is restored when that execution ends. It is therefore not a reliable way to configure a later request or a separate worker process. The [configuration modes documentation](https://www.php.net/manual/en/configuration.changes.modes.php) explains why a setting may be available in one place but not another.

Configuration should be explicit at deployment boundaries:

- use the intended CLI binary and configuration for commands;
- configure FPM globally and per pool for web traffic;
- pass only the required environment into workers;
- validate required settings at startup;
- avoid exposing diagnostic configuration output to untrusted web clients;
- keep secrets out of command arguments where process listings could reveal them.

## Lifetime and State

### CLI lifetime

A one-shot CLI process gives a strong isolation boundary. At exit, its memory is reclaimed by the operating system. This makes batch commands easier to reason about: process state cannot accidentally affect tomorrow's invocation.

That isolation does not make a command safe by itself. A command can still partially write a file, commit some database changes, or send some messages before failing. If it is retried, the operation must be idempotent or must record progress carefully.

### FPM lifetime

An FPM child commonly handles multiple requests during its lifetime. Request-local application state should be rebuilt for each request. Do not rely on a global, static, singleton, or cached object being recreated merely because a previous request ended.

Most ordinary request code is written as if the process ends after the response, because the request lifecycle hides process reuse. That mental shortcut is acceptable only when the code does not retain mutable state across requests. It becomes dangerous with in-process caches, static accumulators, open resources, custom extensions, and libraries that keep references longer than intended.

### Worker lifetime

A long-running worker makes process reuse visible. An object created before the loop can survive many jobs. A static array can grow with every message. A database connection can become stale. A timezone, locale, random generator, or global context can be changed by one job and observed by the next.

A useful worker rule is:

> Treat each message as isolated application work, even though the operating-system process is shared.

This may mean constructing a message-scoped command object, clearing accumulated collections, resetting a unit of work, reconnecting after transient failures, and stopping the process after a bounded lifetime. The correct policy depends on the libraries and resources involved; the key is to choose it deliberately.

## Failure and Recovery

Failure has a different protocol in each context.

For a CLI command, a non-zero exit status tells the caller that the invocation did not complete successfully. Standard error carries diagnostics. A scheduler may retry, alert, or mark the job failed. A command that prints “done” and then exits zero after a partial failure has violated its contract.

For a web request, the application may return a `4xx` or `5xx` response, but that response does not undo side effects already performed. A client, proxy, or browser may retry after a timeout even when the server completed the database transaction. HTTP status design, idempotency keys, transactions, and durable state must be considered together.

For a worker, the central question is whether the message is acknowledged. A process crash before acknowledgment may cause redelivery. Acknowledging before a side effect is durable can lose work. Performing the side effect and then crashing before acknowledgment can duplicate work. The handler therefore needs an explicit delivery and idempotency strategy.

Timeouts deserve special attention. A CLI command may have a scheduler timeout. An HTTP request may be terminated by a proxy or FPM timeout while PHP continues briefly or is killed. A worker may have a visibility timeout after which its message is delivered again. The timeout is part of the contract, not just a performance setting.

## Security

The boundary changes the threat model.

Web input is attacker-controlled by default. Validate structure and semantics, authorize the operation, and encode output for its destination. Never treat a hidden form field, URL parameter, cookie, or uploaded filename as trusted.

CLI input is not automatically trusted. Arguments may come from an untrusted file, a web-triggered scheduler, a deployment variable, or another process. Avoid passing user-controlled strings into a shell. Check file paths and permissions. Be cautious about secrets in arguments because operating systems may expose command lines to other users or monitoring tools.

Workers consume data produced by another system or earlier version of the application. Validate message schemas and assume messages can be duplicated, delayed, reordered, malformed, or replayed. A queue is not an authorization boundary; the worker still needs to enforce the permissions and invariants relevant to the operation.

FPM itself must be protected as infrastructure. Its FastCGI listening endpoint should not be exposed to an untrusted network. The [FPM configuration documentation](https://www.php.net/manual/en/install.fpm.configuration.php) warns that a client able to open the FastCGI connection can control request configuration and potentially execute code.

## Performance and Capacity

The most visible performance difference is often startup and concurrency shape, not the speed of an individual PHP expression.

A one-shot CLI process pays startup cost for each invocation. That may be irrelevant for a nightly command and significant for a command launched once per tiny task. A long-lived worker amortizes startup and boot cost across messages, but it pays in state-management complexity and memory risk.

FPM provides concurrent child processes. Increasing the pool size can improve throughput only while the rest of the system can support the added concurrency. If 50 PHP children all issue expensive database queries, the database may become the bottleneck before PHP does. More children can increase memory use, contention, and tail latency.

Measure the relevant path:

- CLI: startup time, total duration, peak memory, rows or messages processed, exit status;
- FPM: request rate, latency distribution, active and idle children, queueing, timeouts, error rate, and dependency latency;
- workers: throughput, message age, retry rate, processing duration, memory over time, and shutdown/restart counts.

Use a bounded batch or streaming approach when data is large. A CLI process may be isolated per batch, while a worker may need explicit cleanup between batches. Neither context changes the algorithmic cost of loading ten million records into one PHP array.

## Testing

Testing should match the contract being tested.

### Unit-test the shared logic

The application service should accept explicit values and dependencies:

```php
final class GreetingService
{
    public function greet(string $name): string
    {
        $name = trim($name);

        if ($name === '') {
            throw new InvalidArgumentException('Name must not be empty.');
        }

        return "Hello, {$name}!";
    }
}
```

Unit tests can cover valid input, invalid input, normalization, and domain failures without starting PHP-FPM, constructing HTTP globals, or launching a shell.

### Test the adapters as adapters

For a CLI command, test argument parsing, help output, standard output, standard error, and exit codes. Include malformed arguments and failures from dependencies. A subprocess test is useful when the exact PHP binary, environment, working directory, and exit behavior matter.

For a web endpoint, test method and route handling, headers, status codes, serialization, authentication, and the mapping of application exceptions to responses. An HTTP-level test catches mistakes that a unit test of the service cannot, such as output sent before headers or an incorrect content type.

For a worker, test message deserialization, acknowledgment order, retry and rejection behavior, duplicate delivery, malformed messages, and graceful shutdown. A fake broker can make these tests deterministic; a smaller integration suite should verify the real client and serialization boundaries.

Do not make every test an end-to-end test. The shared logic should be quick to test in isolation, while a smaller number of boundary tests verifies the contracts that globals, SAPIs, brokers, and supervisors provide.

## Bad Example

This design hides three incompatible contracts in one function:

```php
function run(): void
{
    $input = $_POST['name'] ?? $argv[1] ?? '';
    $result = doWork($input);

    echo json_encode($result);
    exit(0);
}
```

In a web request, the command-line variable may not exist and the output has no explicit status or content type. In a CLI invocation, JSON may be mixed with warnings or progress messages, and `exit(0)` makes the function difficult to compose or test. In a worker, printing JSON does not acknowledge a message. The code has confused input acquisition, application work, output formatting, and process control.

## Better Example

Keep the service independent and let each entry point own its protocol:

```php
final readonly class WorkInput
{
    public function __construct(public string $value)
    {
    }
}

final class WorkService
{
    public function run(WorkInput $input): array
    {
        return ['value' => trim($input->value)];
    }
}
```

The CLI adapter can parse `$argv`, catch a domain exception, write diagnostics to `STDERR`, and return an exit code. The web adapter can parse the request, call `WorkService`, and create an HTTP response. The worker can deserialize a message, call the same service, and acknowledge only after the required side effect is durable.

The separation does not eliminate environment-specific code. It puts that code where it belongs and prevents every business rule from knowing about every possible environment.

## Common Mistakes

- Assuming the CLI and FPM use the same `php.ini`, extensions, user, or working directory.
- Treating an FPM child process as if it is destroyed after every request.
- Treating a queue worker as if each message starts a fresh process.
- Reading superglobals or `$argv` deep inside application logic.
- Printing diagnostics into a machine-readable response or stdout data stream.
- Returning HTTP status codes from code that does not own the HTTP boundary.
- Using process exit as ordinary application control flow.
- Assuming a successful response means every side effect is durable.
- Acknowledging a message before the operation it represents is safely committed.
- Retrying a command or request without considering duplicate side effects.
- Increasing FPM children without checking memory, database capacity, and tail latency.
- Testing only the domain service and never testing the CLI, HTTP, or worker contract.

## Senior Engineer Thinking

When a feature must run in more than one environment, write down the contract before choosing the implementation:

1. What event starts the work?
2. What input is guaranteed, and what input is untrusted?
3. What output or acknowledgment proves success?
4. What is the process and logical-work lifetime?
5. Which state may survive one unit of work?
6. What happens after a timeout, crash, or retry?
7. Which configuration and permissions apply?
8. Which boundary tests must fail if the contract changes?

This makes trade-offs visible. A nightly import may be simplest as a CLI command because process isolation and exit codes fit the job. A customer-facing operation needs an HTTP boundary because clients need status and response semantics. A slow, retryable task may fit a queue worker, provided its side effects are idempotent and its lifecycle is supervised.

The reusable unit is usually not “a controller that also works in a command.” It is application logic with explicit inputs and outputs, called by a controller, command, or worker adapter that understands its own environment.

## Exercises

1. Take a function that reads `$_POST` or `$_GET` and refactor it so its core logic accepts a typed input object. Add a CLI adapter that supplies the same input from `$argv`.
2. Write a CLI command that processes a file in batches. Record its exit codes, peak memory, and behavior when the file is truncated halfway through.
3. Design a worker handler for a message that sends an email. List the point at which acknowledgment occurs, the possible duplicate-delivery scenario, and the idempotency key you would use.
4. Run the same diagnostic script through the CLI and web runtime. Compare `PHP_SAPI`, the loaded `php.ini`, enabled extensions, current working directory, user permissions, and relevant environment variables.

## Review Questions

1. Why is an FPM child process not the same abstraction as a queue worker?
2. What are the normal input and output channels for a CLI command?
3. Why should application logic not read `$_POST` directly?
4. How does a non-zero CLI exit status differ from an HTTP `500` response?
5. What state can survive after an FPM request ends?
6. Why can a worker process one message successfully and still be unsafe for the next message?
7. Why might increasing `pm.max_children` reduce reliability instead of improving it?
8. Which tests belong at the shared-logic boundary, and which belong at the SAPI or broker boundary?

## Summary

CLI, web/FPM, and application workers use the same PHP language through different execution contracts. CLI programs receive arguments, environment variables, files, and streams, and report success primarily through output and exit status. Web programs receive HTTP requests and return status, headers, and a body through a web server and PHP-FPM. Workers consume messages repeatedly and must coordinate acknowledgment, retries, shutdown, and state isolation.

The decisive difference is lifecycle. A CLI invocation is usually isolated by process exit. An FPM child may serve many requests. A long-running worker handles many logical jobs in one process. Configuration, failure handling, security, capacity planning, and testing must reflect that lifecycle.

Keep environment-specific input and output at adapters. Give shared application logic explicit contracts. Then choose CLI, FPM, or a worker according to the workload's delivery, latency, retry, and operational constraints—not according to habit.

## Sources and Further Reading

- [PHP Manual: Using PHP from the command line](https://www.php.net/commandline)
- [PHP Manual: CLI usage](https://www.php.net/manual/en/features.commandline.usage.php)
- [PHP Manual: FastCGI Process Manager (FPM)](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: Where a configuration setting may be set](https://www.php.net/manual/en/configuration.changes.modes.php)
- [PHP Manual: `ini_set()`](https://www.php.net/manual/en/function.ini-set.php)
