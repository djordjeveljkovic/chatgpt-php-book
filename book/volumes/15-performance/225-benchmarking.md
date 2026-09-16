---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 225
title: Benchmarking
slug: benchmarking
status: complete
summary: ../../_ai/chapter-summaries/225-benchmarking-summary.md
---

# Chapter 225 — Benchmarking

## Why This Matters

A benchmark compares a defined operation under controlled conditions. It can reveal whether a parser, algorithm, serializer, or query strategy changed performance, but it cannot by itself predict a production request. Production includes concurrency, network, cache, database, deployment, and failure behavior.

A useful benchmark is a reproducible experiment with a question, workload, baseline, repetitions, and decision rule. Record enough metadata that a later engineer can understand whether a change is real.

## Define the Question

State the operation and target: “Does a packed loop process 100,000 records with less memory?” or “Does this query keep p95 below 200 ms at one million rows?” Define correctness first. A benchmark that compares two implementations producing different results measures different work.

Specify input size, distribution, warm/cold state, concurrency, environment, PHP version, OPcache/JIT settings where relevant, database engine and indexes, and whether setup is included. Benchmark representative small, medium, and large inputs because an optimization can change asymptotic and constant costs.

## A Simple PHP Harness

For a local CPU or allocation comparison, a minimal harness can record wall time and memory:

~~~php
<?php

declare(strict_types=1);

function runBenchmark(string $name, callable $work, int $iterations): void
{
    $started = hrtime(true);
    $beforeMemory = memory_get_usage(true);

    for ($i = 0; $i < $iterations; $i++) {
        $work();
    }

    $elapsedMs = (hrtime(true) - $started) / 1_000_000;
    $deltaMemory = memory_get_peak_usage(true) - $beforeMemory;

    printf(
        "%s: %.3f ms total, %.3f ms/op, peak delta %d bytes\n",
        $name,
        $elapsedMs,
        $elapsedMs / $iterations,
        $deltaMemory,
    );
}
~~~

This is an educational harness, not a statistical framework. Its loop, garbage collection, CPU frequency, background load, and process state affect results. Benchmark tools such as phpbench can control iterations and report distributions; use the installed tool's documentation and keep the benchmark code separate from production tests.

Do not call a network provider or a destructive operation in a benchmark. Use a local fixture, provider sandbox with explicit limits, or a fake that models latency and failure separately.

## Warm-up and Noise

Warm OPcache, class autoloading, connection pools, and caches separately from cold-start measurements. Run warm-up iterations that are excluded from the result. Repeat enough samples and report median or percentile plus variation rather than the single fastest run.

Pin or record CPU, PHP, extensions, operating system, container limits, and process affinity when a small difference matters. A shared CI host is useful for detecting large regressions but noisy for microsecond claims. Use a dedicated environment for decisions that affect capacity.

## Avoiding Benchmark Traps

Common traps include:

* measuring setup in one variant but not the other;
* optimizing a synthetic input unlike production;
* allowing dead-code elimination or unused results to change work;
* measuring one run and ignoring variance;
* comparing cold and warm cache accidentally;
* using a benchmark to justify a security or consistency regression;
* ignoring memory, allocation, error, and tail behavior;
* running a database benchmark without realistic cardinality and indexes.

Consume or assert the result so an implementation cannot skip the work. Validate outputs before timing or after the run. Keep benchmark fixtures deterministic and synthetic.

## Microbenchmarks and System Benchmarks

A microbenchmark isolates a small operation such as date parsing or array traversal. It is useful for algorithm and language questions. A component benchmark includes serialization, a database, or a filesystem. A load test exercises a complete service under concurrency and reveals queueing, saturation, and tail latency.

Use the smallest benchmark that answers the question, then validate the change at a broader level. A faster loop can be irrelevant if the endpoint spends most time waiting for SQL. A lower median can hide a p99 timeout under contention.

## Database and HTTP Benchmarks

Database benchmarks must include schema, indexes, data distribution, query plan, connection behavior, transaction isolation, and concurrent writers. Run EXPLAIN and measure locks and rows, not only client wall time. HTTP benchmarks need keep-alive, TLS, request mix, payload sizes, cache state, PHP-FPM workers, and downstream behavior.

Never run a load test against production without an explicit traffic and data-safety plan. Use a staging environment that matches capacity, or a controlled canary with clear abort thresholds.

## Benchmark Results and Decisions

Report baseline, candidate, sample count, median and tail, memory, error rate, environment, and statistical uncertainty where useful. A 2 percent change may be noise; a 40 percent change may still be irrelevant if the operation is 1 percent of the request. Tie the result to a service budget or capacity model.

Keep benchmark history with code and environment metadata. Re-run after PHP, extension, database, compiler, hardware, or dependency changes. Delete stale benchmarks that no longer answer a decision; a false dashboard creates more work than it saves.

## Failure and Threat Analysis

* **Dead-code benchmark:** unused result removes work. Consume and verify output.
* **Warm-state bias:** one variant benefits from cache or OPcache. Control order and state.
* **Noise mistaken for signal:** one sample drives a change. Repeat and report variation.
* **Unrealistic data:** tiny or uniform fixtures hide scale. Use representative distributions.
* **Capacity blind spot:** single-user speed improves while concurrency fails. Add load tests.
* **Unsafe test traffic:** benchmark calls real side effects. Use fakes, sandboxes, or controlled staging.
* **Correctness trade-off:** faster code weakens authorization or durability. Keep correctness a hard constraint.

## Testing Benchmarks

Benchmark harnesses should have correctness assertions and a smoke run that completes quickly. Separate performance jobs from ordinary unit tests when their runtime or environment differs. Alert on meaningful trends rather than failing a build on normal host variance.

Use profiling to explain an unexpected result. Benchmark tells you that behavior changed; profiling tells you where time or memory went.

## Exercises

1. Benchmark two implementations of a PHP transformation across three input sizes and report median, p95, and peak memory.
2. Design a database benchmark with one million rows, indexes, query plan, and concurrent writes.
3. Create a warm versus cold benchmark for OPcache or a connection pool and document setup.
4. Define abort thresholds and data-safety controls for a staging HTTP load test.

## Review Questions

1. What makes a benchmark reproducible?
2. Why must correctness be checked before comparing speed?
3. How do microbenchmarks differ from load tests?
4. Which warm-up and environment variables affect PHP results?
5. Why can a lower median hide a capacity problem?
6. What evidence should accompany a performance decision?

## Summary

Benchmarking is controlled comparison, not a stopwatch around arbitrary code. Define the question and workload, control warm state and environment, repeat samples, verify correctness, report distributions and memory, and validate microbenchmark gains with component or load tests. Use profiling to explain the result and never trade correctness for a synthetic speedup.

## References

- [PHP Manual: hrtime](https://www.php.net/manual/en/function.hrtime.php)
- [PHP Manual: memory_get_peak_usage](https://www.php.net/manual/en/function.memory-get-peak-usage.php)
- [PhpBench](https://phpbench.github.io/phpbench/)
- [Apache JMeter](https://jmeter.apache.org/)

