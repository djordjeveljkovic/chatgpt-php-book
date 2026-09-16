# AI Summary — Chapter 66 — Signals

- Status: complete
- Volume: Volume 5 — PHP RUNTIME
- Last updated: 2026-09-14

## Written material

Explains operating-system signals, CLI/`pcntl` scope, synchronous versus asynchronous dispatch, graceful shutdown, worker draining, bounded units of work, escalation, child reaping, and recovery after forced termination.

## Concepts already explained

Signal lifecycle, stop flags, drain budgets, idempotent retry, `SIGTERM`, `SIGINT`, `SIGHUP`, `SIGCHLD`, `pcntl_signal()`, `pcntl_async_signals()`, `pcntl_signal_dispatch()`, and `pcntl_waitpid()`.

## Terminology established

Signal handler, stop request, draining, cleanup boundary, unit of work, forced termination, durable recovery.

## Examples used

Signal-aware CLI loop; draining queue worker; unsafe I/O-heavy handler; lifecycle and failure diagrams.

## Cross-references

Chapter 67 for bounded stream I/O; Chapter 69 for child processes and `SIGCHLD`; earlier worker-process chapters.

## Open threads

Apply signal-aware lifecycle reasoning to streams, processes, and configuration reloads.

## Exact next section

Chapter 67 — Streams: the Why This Matters section.

## Technical verification notes

References use official PHP manuals for `pcntl` and signal APIs. Platform support and extension availability must be checked in the target CLI deployment.
