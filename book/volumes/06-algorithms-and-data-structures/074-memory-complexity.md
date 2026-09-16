---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 74
title: Memory Complexity
slug: memory-complexity
status: complete
summary: ../../_ai/chapter-summaries/074-memory-complexity-summary.md
---

# Chapter 74 — Memory Complexity

## Why This Matters

An algorithm can be fast enough in CPU time and still fail because it needs too much memory. PHP applications are particularly vulnerable to this mistake because arrays are flexible and convenient, while their internal representation carries more overhead than a compact typed vector in a systems language.

Memory complexity asks how much additional live state an algorithm needs as input grows. In a request, that state competes with the framework, loaded objects, decoded payloads, database results, and extensions. In a worker, retained state can survive one job and damage every later job.

The key quantity is often peak live memory, not the final size of the result. A transformation that creates a second full array can briefly require the input and output at the same time.

## Mental Model

```text
process memory
├── input still needed
├── output being built
├── indexes / lookup state
├── temporary values and copies
├── runtime/framework state
└── native allocations and allocator reserve
```

For an algorithm, distinguish:

```text
input space       memory owned by the input supplied to the algorithm
auxiliary space   additional working memory
output space      memory required for the result
peak live space   maximum simultaneous live allocation
```

Textbook space complexity often reports auxiliary space. Operations teams care about peak process memory. Both views are useful, but they answer different questions.

## Core Concept

These two functions both uppercase lines:

```php
/** @return list<string> */
function uppercased(array $lines): array
{
    $result = [];

    foreach ($lines as $line) {
        $result[] = strtoupper($line);
    }

    return $result;
}

function writeUppercased(iterable $lines, string $path): void
{
    $stream = fopen($path, 'wb');
    if ($stream === false) {
        throw new RuntimeException('Unable to open output file.');
    }

    try {
        foreach ($lines as $line) {
            fwrite($stream, strtoupper($line));
        }
    } finally {
        fclose($stream);
    }
}
```

The first returns O(n) output and requires the whole result to remain live. The second keeps only the current line and stream buffers in the algorithm, so its additional working memory is bounded by the largest item and buffering policy. It is not automatically better: it changes the API, output timing, error recovery, and ability to retry from memory.

## How It Works

### Live ranges matter

Memory is used by values that are still reachable or held by the runtime. A variable that goes out of scope can become reclaimable, but allocator reuse means process RSS may not immediately fall. When reasoning about an algorithm, draw the lifetime of the important values:

```text
read input ───────────────┐
build output              ├─ peak: input + output
release input ────────────┘
flush output
```

If the input is transformed one item at a time and the old item is released before the next arrives, the live set can stay bounded. If `array_map()` or `iterator_to_array()` materializes a whole stage, the memory curve changes.

### Hidden duplication

PHP’s copy-on-write can share array or string storage until a mutation requires separation. That is an important optimization, but it is not a promise that every apparent copy is free:

```php
$working = $records;       // may initially share storage
$working[] = $newRecord;   // may separate and allocate
```

A transformation may also create temporary arrays:

```php
$result = array_values(array_filter($records, $predicate));
```

At different points, the input, filtered intermediate, and reindexed result may overlap. Measure peak memory for the actual PHP version and data shape. Chapter 52 covers COW; Chapter 56 covers PHP-managed allocation, allocator reuse, and RSS.

### Recursion and call state

Recursive algorithms consume stack and frame state per active depth. A recursive traversal with depth `d` may use O(d) auxiliary space even if it processes each node once. A hostile deeply nested input can turn recursion into a reliability problem before the total node count appears large.

An explicit stack moves that state into a userland collection. It may make bounds and recovery easier to control, but the memory is still required. Changing recursion to iteration changes where state lives; it does not make the state disappear.

## PHP Collection Costs

The PHP array is an ordered map. It is excellent for many application tasks, but a large array has per-entry overhead for keys, values, buckets, and associated allocation. Therefore “one integer per record” is a logical description, not a reliable byte estimate.

A map keyed by a long user identifier stores the key and the value. A set-like map that uses `true` as every value still stores each key and entry metadata. A nested array multiplies this structure at each level. Objects add instance and property storage. Strings add their own payload and refcounted representation.

Use asymptotic space to compare growth, and use measurement to plan capacity. A `memory_limit` protects PHP-managed allocation, but the process can also consume native memory and resident pages that require operational monitoring.

## Practical Example: Whole File versus Bounded Batches

This common pattern materializes a file:

```php
$rows = file('events.ndjson', FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

foreach ($rows as $row) {
    process(json_decode($row, true, flags: JSON_THROW_ON_ERROR));
}
```

The file contents and every line remain in `$rows` while processing. If `process()` also retains each decoded record, the live set is larger still.

A streaming reader bounds input retention:

```php
function lines(string $path): Generator
{
    $stream = fopen($path, 'rb');
    if ($stream === false) {
        throw new RuntimeException('Unable to open input.');
    }

    try {
        while (($line = fgets($stream)) !== false) {
            yield rtrim($line, "\r\n");
        }
    } finally {
        fclose($stream);
    }
}

foreach (lines('events.ndjson') as $line) {
    process(json_decode($line, true, flags: JSON_THROW_ON_ERROR));
}
```

This reduces materialized input, but `process()` must also release or bound its state. A generator does not help if the consumer appends every decoded record to a global array.

### Bounded batch design

For a batch-oriented sink, retain only a fixed number of records:

```php
$batch = [];

foreach (lines('events.ndjson') as $line) {
    $batch[] = json_decode($line, true, flags: JSON_THROW_ON_ERROR);

    if (count($batch) === 500) {
        persistBatch($batch);
        $batch = [];
    }
}

if ($batch !== []) {
    persistBatch($batch);
}
```

The bound is approximate: one record may be much larger than another, the sink may allocate its own buffers, and an exception may leave diagnostic state alive. Use byte limits as well as item limits when record size varies.

## What PHP Does

`memory_get_usage()` reports memory allocated to the PHP script, while its `real_usage` option includes memory allocated from the system but not currently used according to the manual. These values do not represent every native allocation or the complete process RSS. `memory_get_peak_usage()` is useful for finding peaks, and the `memory_limit` directive constrains PHP allocation, but capacity planning should also inspect worker RSS and container limits.

Memory can become reclaimable without returning to the operating system immediately. Zend MM and the underlying allocator may reuse freed blocks. That is normal; it is also why a long-running worker should be monitored for retained references, fragmentation, extension allocations, and workload-dependent peaks.

## Performance

Time and space trade off against each other:

```text
repeated scan   → less index memory, more CPU
in-memory index → more memory, cheaper repeated lookup
sort-and-merge  → sorting cost, sequential processing
streaming       → low input memory, less random access and harder replay
```

Do not choose the lowest peak memory without considering throughput and recovery. A streaming import may need durable checkpoints because the process no longer has all input available for a retry. A batch size may reduce memory but increase database round trips. A cache may save CPU but consume shared memory and create invalidation complexity.

Measure at the boundary that matters: a CLI import, an FPM worker handling concurrent requests, or a queue worker after many jobs. A short CLI script that exits after each operation will not reveal retention in a long-lived process.

## Security

Memory exhaustion is an availability vulnerability. Limit upload bytes before parsing, cap collection sizes where the parser allows it, bound batches, and reject untrusted input that would create an enormous key set. Do not use `memory_limit = -1` as a fix for an unbounded algorithm.

Be cautious with error reporting. Dumping a large payload in an exception or log can create a second memory spike and expose secrets. Redact and truncate diagnostics.

## Testing

Test memory behavior as a property of the workload:

1. Process empty, small, typical, maximum, and oversized inputs.
2. Measure current and peak memory around each pipeline stage.
3. Run a worker-style test that processes many batches and checks that retained state stays bounded.
4. Include large individual records, not only a large count of tiny records.
5. Test failure midway through a stream and verify file handles, temporary files, and batch state are cleaned up.

Memory thresholds should allow for build and runner variance. The important assertions are often “does not materialize the entire input” and “does not grow with total jobs,” supported by profiling rather than a brittle exact byte count.

## Common Mistakes

- Reporting only the final result size and ignoring peak overlap.
- Assuming `unset()` immediately reduces RSS.
- Treating copy-on-write as a guarantee that mutation is free.
- Converting a generator back to an array before processing it.
- Storing full records when only IDs or counters are needed.
- Retaining closures, logs, caches, or exception objects in a worker.
- Solving a memory limit by raising the limit until the host is exhausted.

## Senior Engineer Thinking

Memory complexity is ownership and lifetime made measurable. Ask what must remain live, for how long, and at which boundary. Then select a representation, streaming model, batch size, and restart policy that fit the real memory budget.

The strongest design can state a bound in domain terms: “at most 500 records and 32 MiB of decoded payload per batch, plus sink buffers,” not merely “it uses a generator.”

## Exercises

1. Compare whole-file, generator, and fixed-byte batch designs for a 5 GB import.
2. Instrument a transformation that uses `array_filter()` and `array_values()`; identify overlapping live values.
3. Build a worker test that processes 10,000 jobs and reports peak memory every 1,000 jobs.
4. Explain the difference between `memory_get_usage()`, peak PHP allocation, RSS, and the container memory limit.

## Review Questions

1. What is the difference between input, auxiliary, output, and peak live space?
2. Why can a linear-time transformation still exceed a memory limit?
3. When does a generator reduce memory, and when does it not?
4. Why should a worker be tested across many jobs rather than one?
5. Which memory costs are not captured by a simple PHP array count?

## Summary

Memory complexity describes how live state grows, but operations care about peak process usage, allocator behavior, and worker lifetime. Distinguish input from auxiliary and output space, account for temporary overlap and COW separation, and use streaming or bounded batches when the data does not fit safely in memory.

## References

- [PHP Manual: `memory_get_usage()`](https://www.php.net/manual/en/function.memory-get-usage.php)
- [PHP Manual: `memory_get_peak_usage()`](https://www.php.net/manual/en/function.memory-get_peak-usage.php)
- [PHP Manual: `memory_limit`](https://www.php.net/manual/en/ini.core.php)
- [PHP Manual: Generators](https://www.php.net/manual/en/language.generators.php)
- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
