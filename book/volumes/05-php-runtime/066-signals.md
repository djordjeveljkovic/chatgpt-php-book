---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 66
title: Signals
slug: signals
status: complete
summary: ../../_ai/chapter-summaries/066-signals-summary.md
---

# Chapter 66 — Signals

## Why This Matters

A worker is not finished merely because its loop has no more work. In production, a supervisor, container runtime, or operator may send `SIGTERM` during a deployment; an interactive user may send `SIGINT`; the operating system may notify a parent that a child process exited. A PHP process that understands none of these events can be killed while holding a lock, halfway through a message, or before it flushes useful telemetry.

Signals are asynchronous notifications delivered by the operating system. They are not exceptions, HTTP requests, or portable messages. In PHP they are primarily useful to CLI programs and long-running workers, through the `pcntl` extension. The engineering problem is to turn an asynchronous notification into a safe, observable state transition.

## Mental Model

```text
supervisor / terminal / kernel
            │ sends signal
            ▼
       PHP process
            │ dispatches handler
            ▼
   small state change (stop = true)
            │ observed at a safe boundary
            ▼
 finish current unit → release resources → exit
```

```text
start → install handlers → run bounded work
                         ↓ signal
                 stop accepting work
                         ↓
              finish/retry current unit
                         ↓
                  cleanup and exit
```

The signal is a request to change process state. It is not permission to perform arbitrary application work inside a handler. A graceful worker usually stops reserving new work, finishes a bounded unit according to policy, releases resources, and exits within a drain budget.

## Core Concept

`pcntl_signal()` associates a signal number with a callable. With asynchronous signal handling enabled, PHP can invoke the callable without requiring an explicit dispatch call. With synchronous handling, the process calls `pcntl_signal_dispatch()` at known points. Explicit dispatch makes interruption points easier to reason about; asynchronous dispatch is convenient when the loop has safe handlers.

The `pcntl` extension is not normally available in web/FPM deployments and is intended for command-line process control. Check the runtime and document supported operating systems. Common lifecycle signals are:

| Signal | Typical meaning | Worker policy |
| --- | --- | --- |
| `SIGTERM` | requested termination | stop taking work and drain |
| `SIGINT` | interactive interruption | usually same graceful path |
| `SIGHUP` | hangup; often repurposed for reload | define explicitly |
| `SIGCHLD` | child state changed | reap children if created |

Signal numbers and availability are platform-dependent. Use PHP’s constants rather than hard-coded integers.

## Minimal Example

```php
<?php

declare(strict_types=1);

if (! extension_loaded('pcntl')) {
    throw new RuntimeException('pcntl is required for this CLI worker.');
}

$stopRequested = false;
pcntl_async_signals(true);

$handler = static function (int $signal) use (&$stopRequested): void {
    $stopRequested = true;
};

pcntl_signal(SIGTERM, $handler);
pcntl_signal(SIGINT, $handler);

while (! $stopRequested) {
    // Poll or receive one bounded unit of work.
    usleep(100_000);
}

fwrite(STDOUT, "Drained; exiting\n");
exit(0);
```

The handler records intent. It does not close a database connection, write a large log entry, or call an external service. The main loop owns cleanup and can stop at a defined boundary.

## How It Works

A signal can arrive while the process is sleeping, blocked in I/O, or executing userland code. A loop should therefore use bounded waits and repeatedly inspect its state:

```text
bounded wait → dispatch/observe → process one unit
      ↑              │                 │
      └──────────────┴────── repeat ───┘
```

The important boundary is the unit of work. “Graceful” does not mean “never interrupt.” It means the application has decided what can be completed, what can be retried, and what must be abandoned. A payment capture and a thumbnail resize do not necessarily have the same shutdown policy.

## Practical Example: A Draining Worker

```php
<?php

declare(strict_types=1);

final class Worker
{
    private bool $stopRequested = false;

    public function run(): int
    {
        pcntl_async_signals(true);
        pcntl_signal(SIGTERM, fn (): bool => $this->requestStop());
        pcntl_signal(SIGINT, fn (): bool => $this->requestStop());

        while (! $this->stopRequested) {
            $job = $this->reserveOneJob();
            if ($job === null) {
                usleep(250_000);
                continue;
            }

            try {
                $this->handle($job);
                $this->acknowledge($job);
            } catch (Throwable $exception) {
                $this->recordFailure($job, $exception);
            }
        }

        $this->closeResources();
        return 0;
    }

    private function requestStop(): bool { $this->stopRequested = true; return true; }
    private function reserveOneJob(): ?array { return null; }
    private function handle(array $job): void {}
    private function acknowledge(array $job): void {}
    private function recordFailure(array $job, Throwable $exception): void {}
    private function closeResources(): void {}
}
```

If termination occurs after reservation but before acknowledgement, the queue must make the job visible again or provide an idempotent retry. Signals cannot solve that consistency problem; they only provide the trigger for orderly shutdown.

## Bad Example and Better Design

```php
pcntl_signal(SIGTERM, static function (): void {
    $database->commit();
    $httpClient->post('/shutdown');
    file_put_contents('/var/log/worker.log', "stopping\n", FILE_APPEND);
});
```

This handler performs blocking I/O, captures mutable state with unclear lifetime, and assumes shutdown work cannot fail. A supervisor may escalate to `SIGKILL`, which cannot be handled. Prefer `handler: stopRequested = true` and `loop: stop reserving → finish bounded job → release → exit`.

Make the drain budget explicit. For example, finish the current idempotent job for at most 30 seconds, then exit so the queue can retry it. Record the signal, current job identifier, elapsed drain time, and final outcome.

## Edge Cases and Failure Modes

- `SIGKILL` cannot be caught. Durable recovery must exist for forced termination.
- A second `SIGTERM` may arrive while draining. Escalate intentionally or shorten the policy.
- If a handler changes a flag between a side effect and acknowledgement, the job may be retried. Design the side effect to be idempotent or transactionally safe.
- Signal handling differs on Windows and Unix-like systems. Test the deployment platform.
- `SIGCHLD` is a notification; it does not itself reap a child. Chapter 69 discusses waiting.
- A blocked stream can delay graceful shutdown. Use timeouts and bounded I/O, as discussed in Chapter 67.

## Performance

Signal handlers should be constant-time in intent: set state, increment a counter, or record a small fact. Shutdown cost is dominated by drained work and cleanup I/O. Avoid hot polling that wastes CPU and unbounded blocking that prevents the loop from observing state. Measure termination latency.

## Security

Only trusted operators or supervisors should signal a worker. Unix permissions, service-manager policy, and container isolation matter. Do not expose a web endpoint that directly signals arbitrary processes without strict authorization. Shutdown is an availability-sensitive control plane.

## Testing

Unit-test that `SIGTERM` sets the stop request and no new job is reserved after the boundary. Integration-test a disposable CLI worker: send `SIGTERM`, assert bounded exit, verify lock release, and verify an unacknowledged job is retryable. Also test forced termination recovery, duplicate delivery, and a signal during slow I/O.

## Common Mistakes

- Assuming FPM application code can use `pcntl` like a CLI worker.
- Doing substantial work inside a handler.
- Calling a worker graceful without defining its unit-of-work boundary.
- Forgetting `SIGKILL`, host failure, and power loss.
- Acknowledging before the side effect is durable.
- Having no drain timeout.

## Senior Engineer Thinking

Ask what work is active, whether it can be retried, which resources must be released, and what durable evidence lets the next process recover. Signals are one input to a lifecycle protocol; leases, transactions, idempotency, cleanup, and observability make that protocol reliable.

## Exercises

1. Add `SIGTERM` handling to a CLI loop and measure signal-to-exit time.
2. Simulate termination after reserving a job but before acknowledgement. Design a retry rule.
3. Compare explicit `pcntl_signal_dispatch()` with asynchronous handling in a polling loop.
4. Make the first signal drain and the second signal exit promptly; document the trade-off.

## Review Questions

1. Why should a handler usually set state instead of performing cleanup?
2. What does graceful shutdown mean for non-idempotent work?
3. Why can a process exit without running cleanup code?
4. What is the relationship between `SIGCHLD` and `pcntl_waitpid()`?
5. Which metrics reveal that workers fail to drain?

## Summary

Signals are asynchronous operating-system notifications, most useful to PHP CLI workers through `pcntl`. Convert them into a small state change, observe it at safe work boundaries, drain within a budget, and rely on durable retry and idempotency for forced termination. The next chapter examines streams used across files, sockets, and process pipes.

## References

- [PHP Manual: Process Control](https://www.php.net/manual/en/book.pcntl.php)
- [PHP Manual: `pcntl_signal()`](https://www.php.net/manual/en/function.pcntl-signal.php)
- [PHP Manual: `pcntl_async_signals()`](https://www.php.net/manual/en/function.pcntl-async-signals.php)
- [PHP Manual: `pcntl_signal_dispatch()`](https://www.php.net/manual/en/function.pcntl-signal-dispatch.php)
- [PHP Manual: `pcntl_waitpid()`](https://www.php.net/manual/en/function.pcntl-waitpid.php)
