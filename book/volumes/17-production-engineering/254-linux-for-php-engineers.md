---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 254
title: Linux for PHP Engineers
slug: linux-for-php-engineers
status: complete
summary: ../../_ai/chapter-summaries/254-linux-for-php-engineers-summary.md
---

# Chapter 254 — Linux for PHP Engineers

## Why This Matters

PHP applications run inside an operating-system environment that supplies processes, memory, files, sockets, signals, users, and limits. A slow or failing request may be caused by Linux scheduling, file descriptors, memory pressure, DNS, disk, or process limits rather than PHP code.

Production debugging improves when an engineer can move from the application symptom to the relevant OS resource without guessing. The goal is not to memorize every command; it is to understand what each observation measures and which boundary owns the fix.

## The Process Model

```text
host or container
  ├─ web server
  ├─ PHP-FPM master → worker processes
  ├─ queue workers
  └─ monitoring and system services
```

A PHP-FPM worker is an operating-system process with its own address space and file descriptors. OPcache shared memory and external services are shared boundaries. A worker's PHP memory limit is not the host's available memory, and a container limit is not necessarily the host's total capacity.

Processes can be created, scheduled, paused, terminated, and restarted. A request that sends a response may still have background work only if the process or queue contract permits it; durable work belongs in a durable boundary.

## CPU and Memory

CPU saturation may appear as high process CPU, a long run queue, increased latency, or throttling in a container. Low PHP CPU does not prove health; workers may be waiting on a database, network, lock, or disk.

Memory pressure can cause allocation failures, swapping, reclaim, or an out-of-memory kill. Measure resident memory per worker, shared memory, page cache, container limits, and host headroom. A process that grows after every queue message needs investigation and a lifecycle policy.

Use the smallest useful observation: process list and elapsed time for a stuck worker, memory metrics for growth, and system-level counters for host pressure. Do not kill processes as a first diagnosis when a safe drain or profile can preserve evidence.

## Files, Sockets, and Descriptors

Files, TCP connections, pipes, logs, and listening sockets consume file descriptors. A descriptor leak can make an application fail to open database connections or write logs even when CPU and memory appear normal.

A descriptor has a process owner and lifecycle. Close response bodies, streams, sockets, and temporary files on every path. Long-running PHP workers need special attention because resources survive between messages.

```text
open descriptors = files + sockets + pipes + event handles
```

The configured process limit applies per process, while the host and container may impose other limits. Compare the application's peak and failure behavior with the actual runtime configuration.

## Users, Permissions, and Filesystems

The service user determines which files and sockets a process can access. Keep code, configuration, writable runtime directories, uploads, and logs separate. A directory that is writable by PHP should not also contain executable application code unless the deployment explicitly requires it.

Filesystem behavior includes permissions, ownership, mount options, available bytes, inode exhaustion, latency, and network storage failure. “Disk full” can mean no bytes or no inodes. Temporary files and session directories need monitoring and cleanup.

Do not run the application as root to hide a permissions problem. Fix ownership, group membership, directory modes, and deployment paths.

## Signals and Shutdown

Unix signals coordinate process lifecycle. A graceful termination signal asks a worker to stop accepting new work and finish or release current work. A forced kill stops execution without application cleanup.

Queue consumers should stop claiming messages on shutdown, finish within a deadline, acknowledge only durable effects, and allow redelivery after forced termination. FPM reload behavior is a deployment contract; observe old-worker age and in-flight requests. See [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md).

## Useful Observations

Commands vary by distribution and container image. Learn the meaning before using one:

```text
process list       → which processes exist and how long they run
CPU/memory view    → current resource use, not root cause by itself
open-file listing  → descriptor ownership and leaks
socket inspection  → listeners, connections, and states
filesystem stats   → bytes, inodes, and mount capacity
system logs        → service failures, kills, and kernel reports
```

Capture timestamps, host or container identity, release, process ID, and relevant limits. A command output without context is difficult to compare with an application trace.

## PHP Runtime Boundary

PHP exposes process, memory, filesystem, stream, and timing functions, but the operating system remains the authority for scheduling and resource limits. `memory_get_usage()` does not replace resident-memory observation. A PHP exception does not explain an external kill. `hrtime()` measures elapsed time; it does not identify which OS resource caused the wait.

Use application metrics and OS metrics together. Correlate PHP-FPM active workers, database pool wait, process memory, CPU pressure, file descriptors, and network errors.

## Containers and Limits

Containers isolate processes and impose namespaces and resource controls; they do not make resource limits disappear. A container may see a CPU or memory quota different from the host and may be terminated when its limit is exceeded. Confirm the runtime's cgroup and orchestrator configuration rather than assuming host values.

Set limits with enough room for PHP workers, OPcache, web server, sidecars, and shutdown behavior. A limit that is too low causes kills; one that is too high can let one workload harm the host or neighbor.

## Security

Minimize service privileges, protect Unix sockets and runtime directories, restrict diagnostic endpoints, and treat process and filesystem information as operationally sensitive. Avoid exposing environment variables, command lines containing secrets, or full paths in public errors.

Use allow-listed commands for operational automation and avoid passing untrusted strings to a shell. Keep logs protected because they may contain request identifiers, paths, and failure details.

## Testing

Test startup with missing configuration, unwritable directories, full temporary storage, exhausted descriptors, memory pressure, slow disks, DNS failure, graceful shutdown, forced termination, and worker recycling. Exercise the same user and group permissions as production.

Run a controlled process and dependency failure drill with a rollback plan. Do not test destructive resource exhaustion on a shared production host.

## Common Mistakes

* Blaming PHP before checking the process, socket, disk, and memory boundaries.
* Confusing PHP memory limit with host or container memory.
* Ignoring file-descriptor and inode exhaustion.
* Running application processes as root.
* Relying on forced kills for normal queue shutdown.
* Reading host metrics when the process is constrained by a container limit.
* Capturing diagnostics without host, process, release, and timestamp context.
* Exposing process lists, environment variables, or command lines publicly.

## Senior Engineer Thinking

Ask which resource is finite, which layer owns its limit, and what evidence survives the failure. Linux knowledge is valuable when it shortens the path from “requests are timing out” to “these workers are waiting for a saturated descriptor or connection budget,” while preserving safe evidence and recovery.

## Exercises

1. Map a PHP-FPM request to its process, memory, socket, file, database, and filesystem resources.
2. Design a diagnostic snapshot that can correlate a slow worker with OS and application metrics.
3. Simulate graceful shutdown of a queue worker and then forced termination. Verify message outcomes.
4. Compare PHP, container, and host memory and file-descriptor limits in a safe test environment.

## Review Questions

* Why can low PHP CPU coexist with high request latency?
* Which resources consume file descriptors?
* Why is forced termination different from graceful shutdown?
* What does a container limit change about metric interpretation?
* Why should writable directories be separated from application code?
* Which context makes an OS diagnostic useful?

## Summary

Linux supplies the process, memory, filesystem, socket, signal, and limit boundaries around PHP. Diagnose across application and OS evidence, distinguish PHP limits from host and container limits, monitor descriptors and inodes as well as CPU and memory, run with minimal privileges, and design graceful shutdown and recovery for workers.

## References

- [Linux manual pages](https://man7.org/linux/man-pages/)
- [PHP manual: Program execution functions](https://www.php.net/manual/en/book.exec.php)
- [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md)
- [Chapter 257 — Containers](./257-containers.md)
