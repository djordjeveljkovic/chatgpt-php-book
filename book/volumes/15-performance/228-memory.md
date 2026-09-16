---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 228
title: Memory
slug: memory
status: complete
summary: ../../_ai/chapter-summaries/228-memory-summary.md
---

# Chapter 228 — Memory

## Why This Matters

Memory limits are capacity limits. A PHP request that materializes a large result, duplicates an array, or retains a response graph can exceed its limit and fail. A long-running worker can grow gradually through retained objects, static caches, resource handles, or unbounded queues, even when each job is small.

Measure peak memory and live process behavior. Reducing memory can improve throughput by avoiding collection pressure and swapping, but a compact representation that causes extra CPU or I/O may not improve the complete operation.

## PHP Memory Model

PHP variables are zvals that refer to values such as strings, arrays, and objects. Arrays are ordered hash tables and generally cost more memory than packed representations in lower-level languages. Copy-on-write can share a value until one variable modifies it, but a seemingly harmless write or transformation can cause a large copy.

The memory limit is per request in common PHP deployments. Process RSS also includes the engine, extensions, allocator, OPcache mappings, and native buffers, so a request can appear below memory_limit while a worker still contributes to host pressure. Measure both application memory and process/container metrics.

~~~php
<?php

declare(strict_types=1);

function currentMemory(string $label): void
{
    error_log(sprintf(
        '%s current=%d peak=%d',
        $label,
        memory_get_usage(true),
        memory_get_peak_usage(true),
    ));
}

function materialize(array $rows): array
{
    currentMemory('before map');

    $result = array_map(
        static fn (array $row): array => [
            'id' => $row['id'],
            'name' => trim($row['name']),
        ],
        $rows,
    );

    currentMemory('after map');

    return $result;
}
~~~

Memory instrumentation should be sampled and should not log sensitive row data. Peak usage is process-specific and can be affected by earlier work in the request; compare controlled scenarios rather than treating one number as a universal cost.

## Stream Instead of Materialize

If a consumer needs one item at a time, use a generator, cursor, chunked query, or stream. This reduces peak live data from O(n) toward O(1) application storage, although the database, network, and consumer still have buffers and total work.

~~~php
<?php

declare(strict_types=1);

/** @return Generator<int, array<string, mixed>> */
function rows(PDOStatement $statement): Generator
{
    while (($row = $statement->fetch(PDO::FETCH_ASSOC)) !== false) {
        yield $row;
    }
}

function exportCsv(PDOStatement $statement, $output): void
{
    foreach (rows($statement) as $row) {
        fputcsv($output, [$row['id'], $row['name']]);
    }
}
~~~

A streaming response needs output and proxy buffering configured, and a database cursor must have a defined transaction and connection lifetime. Do not call fetchAll before passing rows to the generator. A generator does not make a downstream filesystem or network sink infinitely fast; apply backpressure and size limits.

## Data Structures and Copies

Avoid repeated transformations that retain both the source and all intermediate arrays. Process a list once when possible, use a map for needed lookup, and release references to large values when a scope retains them longer than necessary. Do not unset variables as a ritual; measure the retained graph and simplify its ownership.

Pass-by-value syntax does not necessarily copy immediately because of copy-on-write. Passing an array to a function that modifies it can force a copy. Objects are handles to object state, so references to one object can retain a large graph. Be careful with closures that capture request or job objects and with static caches in long-running processes.

## Long-Running Workers

A worker that handles many jobs must bound every retained collection, close files and sockets, clear request-scoped containers, and reset instrumentation context. Recycle workers after a bounded number of jobs or memory threshold as a safety valve, but find the leak rather than relying only on recycling.

Use a per-job try/finally for cleanup:

~~~php
<?php

declare(strict_types=1);

function runJob(callable $job, Logger $logger): void
{
    $context = [];
    try {
        $context['started_at'] = microtime(true);
        $job();
    } finally {
        $context = [];
        $logger->clearContext();
    }
}
~~~

The cleanup API is application-specific. A framework container, ORM identity map, static cache, or event dispatcher may retain state outside local variables. Reset those components using their supported lifecycle hooks.

## Database and HTTP Payloads

Select only required columns, paginate or chunk large reads, and avoid hydrating full models for exports that need three fields. A query that returns 100,000 rows can consume memory in PHP, database buffers, and the client. Move aggregation to the database when its semantics and indexes fit the operation.

Bound JSON request depth and body size, stream file uploads, and avoid decoding a multi-gigabyte document into one array. Response serialization also materializes data; use pagination or a streaming format where clients can process records incrementally. Compression trades CPU for fewer bytes and can increase transient buffers, so measure.

## Garbage Collection and Native Resources

PHP's garbage collector addresses cyclic references, but it does not make unbounded application references disappear. A reference cycle retained by a global or static remains reachable. Destructors and resource wrappers can delay release; close files, database cursors, and network handles deliberately.

A memory increase after a request may be allocator behavior rather than a leak, while a steadily growing live object count suggests retention. Compare heap snapshots or object counts when available and inspect workers over many jobs.

## Testing and Operations

Test large and boundary inputs: maximum page size, long strings, nested documents, many relationships, and batch sizes. Assert that a streaming path does not call a materializing method and that a worker resets state between jobs. Performance tests should record peak PHP memory, process RSS, duration, and output correctness.

Monitor memory per route and worker, peak request usage, worker restarts, out-of-memory kills, batch size, response size, and queue age. Alert on trends rather than a single noisy sample. A memory limit error needs a bounded user-visible response and a diagnostic path that does not log the payload.

## Common Mistakes

- Calling fetchAll or get on an unbounded dataset.
- Assuming copy-on-write prevents all copies after mutation.
- Keeping request data in static caches or long-running worker services.
- Treating memory_limit as the complete process memory budget.
- Adding unset calls without measuring ownership and retention.
- Streaming data while proxy or output buffers still accumulate it.
- Relying on worker recycling without investigating growth.

## Senior Engineer Thinking

Memory performance is about the lifetime and ownership of data. Measure PHP usage and process RSS, stream or chunk when consumers allow it, select compact projections, bound queues and caches, reset long-running workers, and test peak behavior on realistic datasets.

## Exercises

1. Compare a fetchAll export and a cursor-based export at several row counts.
2. Find a long-running worker cache and define a bound, eviction policy, and reset point.
3. Design a streaming upload and export path with limits, cleanup, and proxy behavior.
4. Measure whether an array transformation creates a material copy under your target PHP version.

## Review Questions

1. What does copy-on-write optimize, and when can a copy still occur?
2. Why can process RSS exceed PHP's memory usage?
3. When does a generator reduce memory?
4. Which resources must a long-running worker reset?
5. Why are memory limits and worker recycling not leak fixes?

## Summary

PHP memory performance depends on data lifetime, copy-on-write behavior, array and object representation, payload size, and worker retention. Measure peak usage and process RSS, stream or chunk large work, select minimal data, bound caches and queues, reset long-running state, and test memory at realistic sizes.

## References

- [PHP manual: Memory usage functions](https://www.php.net/manual/en/function.memory-get-usage.php)
- [PHP manual: Generators](https://www.php.net/manual/en/language.generators.php)
- [PHP manual: Garbage Collection](https://www.php.net/manual/en/features.gc.php)
- [PHP manual: Memory limit](https://www.php.net/manual/en/ini.core.php)

