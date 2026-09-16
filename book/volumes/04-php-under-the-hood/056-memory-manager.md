---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 56
title: Memory Manager
slug: memory-manager
status: complete
summary: ../../_ai/chapter-summaries/056-memory-manager-summary.md
---

# Chapter 56 — Memory Manager

## Why This Matters

PHP makes memory management feel invisible. A variable comes into existence when a value is assigned, an array grows when an element is appended, and most values disappear when a request ends. That convenience is a language feature backed by several runtime mechanisms, not evidence that memory is free.

Memory becomes an engineering concern when a request has to decode a large JSON document, when a worker processes thousands of jobs, or when a PHP-FPM pool is sized against a machine's RAM. A process can be killed by the operating system even though PHP's `memory_limit` appears generous. Conversely, a request can hit `memory_limit` while the operating system still has available memory. These are different boundaries.

The useful question is not “does PHP have garbage collection?” It is:

> Which component allocated this memory, how long should it live, which limit observes it, and which process will pay for it?

## Mental Model

There are at least four layers to keep separate:

```text
PHP value and language operation
    ↓
Zend Engine allocation (request or persistent)
    ↓
Process address space and resident pages
    ↓
Operating-system and container limits
```

The Zend memory manager (Zend MM) is the allocator used by the engine for most PHP-managed memory. It obtains larger regions from the system allocator and serves smaller allocations from its own structures. The exact bins, chunks, and bookkeeping are implementation details and can change between PHP releases and build configurations. The stable operational idea is that engine allocations have a lifetime and an accounting path.

The `zval` described in Chapter 47 is a value container. It may contain a scalar directly or point to a separately allocated `zend_string`, array, object, or other payload. Therefore `memory_get_usage()` is not “the number of variables”; it is an observation of memory tracked by the current PHP process under a particular allocator/build.

## Core Concept: Request and Persistent Allocation

Zend-facing C code conventionally distinguishes request-lived allocation from persistent allocation. Conceptually:

```c
void *request_memory = emalloc(size);     /* released with the request */
void *long_lived = pemalloc(size, 1);     /* persistent when requested */
```

This is illustrative API usage, not a recommendation to write an extension without reading the target release's API documentation. Request allocation is appropriate for a value used during one request. Persistent allocation is appropriate for module configuration or data that must survive request shutdown in a server process. A persistent allocation is not automatically shared between FPM workers; each worker is a separate process.

At the PHP level, the practical lifetimes are:

| Lifetime | Typical owner | Example |
| --- | --- | --- |
| Expression or call | VM temporary/call frame | A computed integer or temporary string |
| Request | Engine and userland state | An array returned by a controller |
| Worker process | Persistent extension or process state | An extension cache, depending on design |
| Host/container | OS page cache or external service | File cache, Redis, database |

Request shutdown releases userland state, but the operating system does not necessarily receive every page immediately. The allocator may retain reusable pages for the process. This is one reason resident set size (RSS) and `memory_get_usage()` are related but not interchangeable.

## How It Works

For a small allocation, the conceptual path is:

```text
array append / string creation / object creation
    → engine asks Zend MM for memory
    → Zend MM reuses a suitable free block or obtains memory from the OS
    → payload and bookkeeping are initialized
    → refcounted payload is released or retained as references change
```

Zend MM generally tries to avoid an operating-system call for every PHP value. Reusing blocks reduces system-call overhead, but it also means that freed memory may remain mapped into the process for later allocations. Large allocations can take a different path from small pooled allocations. Exact thresholds are implementation details; do not build capacity calculations around a magic cutoff copied from one source revision.

The allocator must also satisfy alignment, metadata, overflow, and failure requirements. An allocation whose size calculation overflows can become a security bug in C code. Internal code checks sizes before multiplication or addition and fails in a controlled way when memory cannot be obtained.

## What PHP Does

PHP exposes useful observations, not a complete heap profiler:

```php
<?php

declare(strict_types=1);

function reportMemory(string $label): void
{
    printf(
        "%s: current=%d bytes, real=%d bytes, peak=%d bytes\n",
        $label,
        memory_get_usage(false),
        memory_get_usage(true),
        memory_get_peak_usage(true),
    );
}

reportMemory('start');

$rows = array_fill(0, 100_000, ['status' => 'pending']);
reportMemory('after rows');

unset($rows);
reportMemory('after unset');

if (function_exists('gc_mem_caches')) {
    gc_mem_caches();
}

reportMemory('after allocator cache cleanup');
```

`false` requests memory currently used by PHP's memory manager; `true` requests the amount allocated from the system, as documented for the function. The values are diagnostic approximations from the current process, not a portable measure of the memory cost of a data structure. `memory_get_peak_usage()` is process/request-local and only becomes meaningful when sampled in the same lifetime as the work being measured.

`memory_limit` is a PHP configuration limit. A failure to allocate can produce a fatal error, so code should not assume it can catch every out-of-memory condition as an ordinary `Throwable`. The limit does not turn a machine with insufficient RAM into a safe environment, and native libraries or extensions may have allocations whose accounting differs from ordinary userland allocations.

## Minimal Example: Peak Memory Is a Constraint

Reading a whole file is simple:

```php
$contents = file_get_contents($path);
$lines = explode("\\n", $contents);
```

It can require the file contents, the array of line strings, array metadata, and temporary copies at the same time. A streaming alternative bounds application memory:

```php
<?php

declare(strict_types=1);

$handle = fopen($path, 'rb');
if ($handle === false) {
    throw new RuntimeException('Unable to open input');
}

try {
    while (($line = fgets($handle)) !== false) {
        processLine(rtrim($line, "\\r\\n"));
    }
} finally {
    fclose($handle);
}
```

The streaming version still has buffers, a current line, and whatever `processLine()` retains. Its space complexity is approximately O(line length plus working state), rather than O(file size). It can be slower or less convenient, but the constraint is explicit.

## Practical Example: Copy-on-Write and Allocation Shape

PHP arrays are ordered maps, not compact C-style vectors. Chapters 49 and 50 explain their HashTable representation in more detail. This matters here because an array containing one hundred thousand small records can consume far more memory than the corresponding raw bytes in a file.

Copy-on-write can make an assignment cheap initially:

```php
$original = range(1, 100_000);
$alias = $original;       // commonly shares the payload initially

$alias[0] = -1;           // mutation separates the shared value
```

The language semantics are “assignment gives another value”; the sharing and separation are Zend implementation techniques. A benchmark that measures only the assignment misses the later separation cost. The same reasoning applies to function arguments and return values, subject to references and the particular value type.

When memory is tight, change the data flow before attempting micro-optimizations:

1. Select only fields needed for the next operation.
2. Process records in batches and release the batch before reading the next one.
3. Stream files and database cursors where the client supports it.
4. Avoid retaining closures, logs, or result arrays accidentally.
5. Measure peak usage with production-shaped data.

## Production Example: PHP-FPM Capacity

Suppose a worker normally has an RSS of 45 MB and a large report request peaks at 180 MB. A pool of 20 workers cannot safely be budgeted as 20 × the smallest request. A rough first bound is:

```text
pool memory ≈ workers × worst representative worker RSS
             + web server + opcache shared memory + safety margin
```

The formula is a planning approximation. OPcache shared memory is shared among workers on the host, while ordinary request allocations are paid by each worker. Copy-on-write after a process fork and allocator behavior further complicate exact accounting. Measure worker RSS under realistic concurrency and include the container limit, database client buffers, extensions, and sidecars.

`pm.max_children` is therefore a memory and latency decision, not merely a setting to increase when requests queue. Too many children can cause swap, OOM kills, or cascading latency. Too few can underutilize the host. The PHP-FPM process model is covered in Volume V; this chapter supplies the memory boundary needed to reason about it.

## Bad Example

```php
$all = [];
foreach (fetchMillionsOfRows() as $row) {
    $all[] = transform($row);
}
return json_encode($all, JSON_THROW_ON_ERROR);
```

This has several possible retention points: the source iterator may buffer, `$all` retains every transformed row, JSON encoding may create another large string, and the response layer may buffer it again. “The database returned an iterator” does not prove the complete pipeline is streaming.

## Better Example

Design a bounded batch boundary and emit or persist each batch:

```php
function exportInBatches(iterable $rows, int $batchSize, callable $sink): void
{
    if ($batchSize < 1) {
        throw new InvalidArgumentException('Batch size must be positive');
    }

    $batch = [];
    foreach ($rows as $row) {
        $batch[] = transform($row);

        if (count($batch) === $batchSize) {
            $sink($batch);
            $batch = [];
        }
    }

    if ($batch !== []) {
        $sink($batch);
    }
}
```

This does not guarantee a fixed RSS: `transform()` and `sink()` may retain state, and the allocator may keep freed pages. It does establish a testable upper bound on this function's retained batch data and gives operations a knob to tune.

## Edge Cases and Failure Modes

### `unset()` is not a promise about RSS

`unset($value)` removes a variable binding. If it was the last reference to a refcounted value, the value can be released. The allocator may reuse the freed block instead of returning pages to the OS. On a long-running process, observe both logical usage and RSS.

### A reference can extend a lifetime

The `foreach ($items as &$item)` form creates a reference that remains in `$item` after the loop. An accidental later assignment can mutate the final element and keep a relationship alive. `unset($item)` after a by-reference loop is a small but important hygiene step.

### Native memory changes the picture

Image processing, compression, database clients, and custom extensions may allocate outside ordinary userland structures. The extension's documentation and process RSS are necessary evidence; a single `memory_get_usage()` sample is not enough.

### Fragmentation is workload-dependent

Repeatedly allocating differently sized blocks can leave free memory that is difficult to reuse for a particular request. Do not infer fragmentation from one before/after sample. Compare allocation patterns, worker RSS over time, and allocator diagnostics where available.

## Performance

Memory has time costs. Allocation, initialization, copying, cache misses, serialization, and garbage-collector scans all consume CPU. A lower-allocation implementation is not automatically faster if it performs excessive I/O or repeated transformations.

For a meaningful comparison, record input cardinality, PHP version, SAPI, extensions, `memory_limit`, allocator/build, wall time, CPU time where available, current memory, peak memory, and process RSS. Warm up OPcache when comparing web requests, and run enough repetitions to avoid explaining startup noise as application behavior.

## Security

Memory exhaustion is an availability vulnerability. Attackers can submit oversized JSON, deeply nested input, huge multipart forms, or parameters that trigger expensive expansions. Apply request-size limits at the web server and application boundaries, validate before materializing, and use streaming parsers for untrusted large inputs where appropriate.

Do not expose an unrestricted diagnostic endpoint that reports OPcache or allocator details. Memory failures can also leave sensitive data in logs or crash dumps. Treat `memory_limit` as one layer of defense, not input validation.

## Testing and Verification

Test the shape of the data flow, not an exact byte count that will vary across builds:

```php
$peakBefore = memory_get_peak_usage(true);

exportInBatches($fixtureRows, 100, static function (array $batch): void {
    // Assert or record the batch, then let it go out of scope.
});

$peakAfter = memory_get_peak_usage(true);
assert($peakAfter >= $peakBefore);
```

For a useful regression test, run a bounded fixture and assert that peak memory stays below a deliberately generous project budget. Add a larger integration test in a controlled environment. Use a subprocess when testing fatal memory-limit behavior so the test runner itself survives.

Verification checklist:

- Compare `memory_get_usage(false)` and `memory_get_usage(true)` at named phases.
- Observe PHP-FPM worker RSS, not only application counters.
- Repeat with realistic row sizes and maximum accepted input.
- Test a long-running loop for retained references and caches.
- Check behavior near `memory_limit` without depending on a recoverable exception.
- Record PHP version, SAPI, extensions, and configuration with the result.

## Common Mistakes

- Treating PHP arrays as cheap vectors.
- Assuming `unset()` immediately returns memory to the OS.
- Raising `memory_limit` to conceal an unbounded data flow.
- Multiplying per-request memory by workers while forgetting shared and native memory.
- Measuring only the first request or only CLI behavior.
- Assuming an iterator means every downstream stage is streaming.

## Senior Engineer Thinking

When someone says “this endpoint uses 500 MB,” ask what was measured: PHP-managed usage, allocator-reserved memory, worker RSS, or container usage? Then identify the phase and retention path. A capacity decision needs a distribution of measurements under concurrency, not a single number from a developer laptop.

The most durable optimization is often a change in ownership and lifetime: process one record instead of retaining all records, move aggregation to the database, or isolate a risky workload in a worker that can be recycled. Understand the allocator, but design the data flow so the allocator has less work to do.

## Exercises

1. Build a script that reads a large fixture with `file_get_contents()` and with `fgets()`. Compare peak memory and wall time at three file sizes.
2. Change a report exporter from an accumulating array to batches. Test what happens when the sink serializes each batch and when it intentionally retains every batch.
3. Run a long-lived loop that creates and releases arrays. Track logical memory, real allocated memory, and process RSS; explain any divergence.
4. Estimate a safe PHP-FPM pool size from measured worker RSS and a container memory limit. State the safety margin and what was excluded.

## Review Questions

1. Why are `memory_get_usage()` and process RSS different observations?
2. What is the difference between request-lived and persistent allocation?
3. Why can `unset()` release a value without reducing RSS?
4. How can copy-on-write defer rather than eliminate a memory cost?
5. Why is a PHP-FPM pool-size change also a memory-capacity decision?
6. Which input or pipeline changes reduce peak memory without merely raising a limit?

## Summary

Zend MM manages much of the memory used by the engine, but PHP values, allocator-reserved pages, process RSS, and OS/container limits are distinct layers. Request and persistent lifetimes matter, PHP arrays have substantial representation overhead, and copy-on-write changes when a cost appears. Measure phases and worker RSS, bound data flow with streaming or batching, and treat memory exhaustion as an operational and security failure mode.

## Official References

- [PHP `memory_get_usage()`](https://www.php.net/manual/en/function.memory-get-usage.php)
- [PHP `memory_get_peak_usage()`](https://www.php.net/manual/en/function.memory-get-peak-usage.php)
- [PHP `memory_limit`](https://www.php.net/manual/en/ini.core.php#ini.memory-limit)
- [PHP source: Zend memory allocator](https://github.com/php/php-src/blob/master/Zend/zend_alloc.c)
- [PHP source: Zend memory manager](https://github.com/php/php-src/tree/master/Zend)
