---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 148
title: Command Injection
slug: command-injection
status: complete
summary: ../../_ai/chapter-summaries/148-command-injection-summary.md
---

# Chapter 148 — Command Injection

## Why This Matters

Shell commands are interpreted programs. Concatenating request data into a shell command lets an attacker add operators, redirect output, expand files, or execute another command. In PHP, `shell_exec()`, `exec()`, backticks, and process APIs can all become dangerous when the command structure is influenced by untrusted input.

## Mental Model

```text
untrusted value → validation/allow-list → fixed operation → controlled process
```

The safest design is to avoid a shell and use a library or a direct PHP implementation. If an external executable is required, keep the executable and argument structure fixed, pass arguments through an API that does not invoke a shell where possible, and run with least privilege.

## Vulnerable and Safer Shapes

This is unsafe:

```php
$file = $_POST['file'];
$output = shell_exec('pdfinfo ' . $file);
```

Escaping reduces some shell metacharacter risk but does not turn an arbitrary path into an authorized file or remove dangerous program behavior. Prefer a fixed path resolved from a server-owned record and a process API with an argument array when the extension/library supports it:

```php
<?php

declare(strict_types=1);

function inspectPdf(string $storedPath): string
{
    if (!is_file($storedPath) || !is_readable($storedPath)) {
        throw new RuntimeException('File unavailable');
    }

    $command = ['pdfinfo', $storedPath];
    $process = proc_open(
        $command,
        [1 => ['pipe', 'w'], 2 => ['pipe', 'w']],
        $pipes,
        null,
        null,
        ['bypass_shell' => true],
    );
    if (!is_resource($process)) {
        throw new RuntimeException('Could not start inspector');
    }

    $stdout = stream_get_contents($pipes[1]);
    fclose($pipes[1]);
    fclose($pipes[2]);
    $exitCode = proc_close($process);
    if ($exitCode !== 0) {
        throw new RuntimeException('Inspector failed');
    }

    return $stdout;
}
```

Check the deployed PHP version and `proc_open()` behavior for array commands; use a well-maintained process component when it provides clearer cross-platform semantics. `bypass_shell` is not a substitute for path authorization or resource limits.

## Input and Environment Controls

Map a client choice such as `format=pdf` to a fixed executable and argument set. Do not allow arbitrary flags, environment variables, working directories, or executable paths. Resolve a stored object ID to a path in application code, verify ownership, and pass the resolved path as data. Set process timeouts, output-size limits, memory/CPU limits, and a restricted environment where supported.

Run workers under a dedicated operating-system account with no write access to application code or secrets. A command that processes attacker-controlled files should be isolated further when its parser has a history of vulnerabilities.

## Testing and Monitoring

Test spaces, quotes, shell metacharacters, newlines, Unicode, long values, invalid paths, non-zero exits, timeouts, and oversized output. Test that the invoked executable and arguments are the expected fixed policy, not merely that a happy-path string is returned.

Log an operation ID, executable identity, duration, exit category, and bounded diagnostic. Never log secrets or arbitrary command text containing user data. Alert on unexpected executable failures, timeouts, and attempts to select unsupported operations.

## Common Mistakes

- Concatenating user input into a shell command.
- Trusting `escapeshellarg()` as authorization or path validation.
- Allowing user-controlled flags or executable names.
- Running a parser with application privileges and unlimited resources.
- Returning raw stderr to a client.
- Forgetting that environment variables and working directories can affect a process.

## Senior Engineer Thinking

Treat process execution as a privileged integration boundary. Eliminate the shell when possible, fix the operation shape, validate the resource, isolate the worker, bound execution, and observe failures. Security depends on the whole process policy, not one escaping call.

## Exercises

1. Replace a shell-based image conversion command with a library or argument-array process call.
2. Design an allow-list for three supported document operations.
3. Add timeout and output limits to a process worker and test abnormal exits.

## Review Questions

1. Why is shell escaping not a complete command-execution policy?
2. Which parts of a process invocation must remain server-controlled?
3. What least-privilege controls reduce impact if a parser is compromised?
4. Which process failures should be observable and bounded?

## Summary

Avoid shell execution for attacker-influenced work. Use libraries or fixed argument-array process calls, validate server-owned resources, restrict executable and environment choices, isolate privileges, bound time and output, and test hostile values and failures.

## References

- [OWASP: OS Command Injection Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html)
- [PHP Manual: `proc_open`](https://www.php.net/manual/en/function.proc-open.php)
- [PHP Manual: Program execution](https://www.php.net/manual/en/book.exec.php)
