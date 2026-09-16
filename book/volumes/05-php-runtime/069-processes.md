---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 69
title: Processes
slug: processes
status: complete
summary: ../../_ai/chapter-summaries/069-processes-summary.md
---

# Chapter 69 — Processes

## Why This Matters

Running another program can be the right boundary for a compiler, image tool, backup utility, or isolated batch task. It can also turn a harmless-looking feature into command injection, a pipe deadlock, an orphaned child, or process exhaustion. PHP exposes several APIs with different behavior; choosing one by habit is not a process design.

Ask who owns the child, how arguments are transferred, how standard input/output/error are connected, how timeouts are enforced, and what the exit status means.

## Mental Model

```text
PHP parent
   │ spawn with explicit argv and descriptors
   ▼
child process ──stdout/stderr──▶ PHP streams
   │
   └── exit code / signal / status
```

```text
validate → build argv/environment → spawn
        → write bounded stdin; drain output
        → observe until exit/deadline
        → terminate by policy
        → collect status; close pipes; reconcile
```

Creating a process is not the same as running a string in a shell. A shell adds expansion, redirection, pipelines, and another injection surface.

## Choosing an API

`exec()`, `system()`, `passthru()`, and `shell_exec()` are shell-oriented conveniences with different output and return-value behavior. `escapeshellarg()` quotes one argument for a shell; it does not make an unsafe operation safe.

`proc_open()` is the general-purpose choice when code must control pipes, environment, working directory, and lifecycle. On supported versions and platforms, an array command form can avoid shell parsing; verify the exact behavior used by the deployment. `pcntl` provides Unix process-control primitives such as `fork()` and `waitpid()`, but forking inside a web worker holding framework resources is specialized.

## Minimal Example

```php
<?php

declare(strict_types=1);

function runVersionCommand(): string
{
    $descriptors = [
        1 => ['pipe', 'w'],
        2 => ['pipe', 'w'],
    ];
    $process = proc_open(['php', '-v'], $descriptors, $pipes);
    if (! is_resource($process)) {
        throw new RuntimeException('Unable to start child process.');
    }

    try {
        $stdout = stream_get_contents($pipes[1]);
        $stderr = stream_get_contents($pipes[2]);
        if ($stdout === false || $stderr === false) {
            throw new RuntimeException('Unable to read child output.');
        }
    } finally {
        foreach ($pipes as $pipe) { fclose($pipe); }
        $status = proc_close($process);
    }

    if ($status !== 0) {
        throw new RuntimeException("Child failed: {$stderr}");
    }
    return $stdout;
}
```

This small example has a weakness: reading stdout fully before stderr can deadlock if the child fills stderr while PHP waits for more stdout. Production code drains both channels concurrently or redirects one deliberately.

## Pipes and Deadlocks

```text
child fills stderr pipe
       ↓
child blocks and stops reading/writing
       ↓
parent waits for stdout or writes stdin
       ↓
neither side makes progress
```

Use `stream_select()` to read whichever output is ready, close stdin when input is complete, and enforce a deadline. Redirect output to controlled files or `/dev/null` only when diagnostics are intentionally unnecessary. Preserve stderr when it is needed for diagnosis.

## Practical Example: Argument-Driven Conversion

```php
<?php

declare(strict_types=1);

function converterCommand(string $input, string $output): array
{
    if (! is_file($input) || ! is_readable($input)) {
        throw new InvalidArgumentException('Input is not readable.');
    }
    $binary = '/usr/local/bin/document-converter';
    if (! is_executable($binary)) {
        throw new RuntimeException('Converter is not installed.');
    }
    return [$binary, '--input', $input, '--output', $output];
}
```

The command is data, not a concatenated shell string. The application still constrains which input and output paths are allowed: safe argument separation does not authorize access to sensitive files.

## Timeouts and Termination

`proc_get_status()` reports process state; it does not impose a timeout. Track a deadline, drain output, and decide whether to request termination. `proc_terminate()` sends a termination request where supported; a child may ignore it or take time to exit. A supervisor may need escalation, but forceful termination risks partial work. Use temporary output plus atomic publication for non-idempotent artifacts.

```text
before deadline → drain and poll
deadline        → graceful termination request
grace expired   → forceful policy / supervisor escalation
always          → collect status, close pipes, reconcile output
```

Exit code is only one result. Record stdout/stderr policy, duration, timeout, termination information when available, and a failure classification.

## Bad Example and Better Design

```php
$name = $_POST['filename'];
shell_exec("convert {$name} /tmp/out.png");
```

The input can alter shell syntax, output is ignored, the binary is not pinned, there is no timeout, and CPU/memory are unbounded. Even escaping the argument would not solve authorization or resource policy.

Instead use an allowlisted operation, generated paths, an explicit argument vector, restricted environment/working directory, OS limits, and a worker boundary for long jobs. Record `pending`, `running`, `succeeded`, and `failed`; publish output only after successful exit and validation.

## Security

Avoid shells. Pin executable paths, pass explicit arguments, allowlist options, use a restricted working directory and environment, apply OS/container limits, and run under a low-privilege account. Validate input/output paths. Treat child output as untrusted. Do not put secrets in command-line arguments if process listings can expose them; use controlled stdin or a documented secret mechanism.

## Performance

Process creation consumes startup time, memory, file descriptors, and process-table capacity. One child per HTTP request can exhaust a host. Prefer a bounded worker pool or queue for repeated expensive commands. Stream large input/output. Measure queue wait, child runtime, CPU, RSS, I/O, exit class, and timeout rate.

## Testing

Test success, nonzero exit, missing binary, malformed output, partial output, slow output, large stderr, timeout, termination, and cleanup after exceptions. Use a deterministic fixture executable for unit tests. Integration-test the actual user, working directory, descriptors, environment, and limits.

## Common Mistakes

- Concatenating untrusted values into shell commands.
- Believing escaping fixes path or operation policy.
- Reading stdout fully before stderr.
- Ignoring exit status.
- Starting unbounded children from request handlers.
- Forgetting `proc_close()`.
- Treating timeout as proof that the child did nothing.

## Senior Engineer Thinking

Treat a child process like a remote dependency on the same host: define a protocol, deadline, resource budget, result schema, retry rule, and observability. “Local” does not mean instantaneous, reliable, or safe.

## Exercises

1. Write a `proc_open()` runner that drains stdout and stderr without deadlock.
2. Add a deadline and classify success, nonzero exit, timeout, and termination.
3. Replace a shell command with an explicit argument vector and allowlisted input.
4. Design a queue-backed conversion workflow that survives parent and child crashes.

## Review Questions

1. Why can two correct-looking pipe reads deadlock?
2. When is `proc_open()` preferable to `shell_exec()`?
3. Why does a safely separated argument still require path authorization?
4. What should happen to partial child output after timeout?
5. Which process resources need an operational budget?

## Summary

Process execution is a lifecycle and security boundary. Prefer explicit arguments and `proc_open()` when control is required, drain pipes without deadlock, enforce deadlines and resource limits, collect status and diagnostics, and reconcile partial output. The next chapter narrows the boundary to environment variables inherited by a PHP process.

## References

- [PHP Manual: Program Execution Functions](https://www.php.net/manual/en/book.exec.php)
- [PHP Manual: `proc_open()`](https://www.php.net/manual/en/function.proc-open.php)
- [PHP Manual: `proc_get_status()`](https://www.php.net/manual/en/function.proc-get-status.php)
- [PHP Manual: `proc_terminate()`](https://www.php.net/manual/en/function.proc-terminate.php)
- [PHP Manual: Process Control](https://www.php.net/manual/en/book.pcntl.php)
