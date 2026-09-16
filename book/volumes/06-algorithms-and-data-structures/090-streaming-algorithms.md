---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 90
title: Streaming Algorithms
slug: streaming-algorithms
status: complete
summary: ../../_ai/chapter-summaries/090-streaming-algorithms-summary.md
---

# Chapter 90 — Streaming Algorithms

## Why This Matters

A PHP command may need to inspect hundreds of millions of log records, one database cursor may represent a data set too large to load into memory, and a worker may observe events indefinitely. In each case, retaining every input is either too expensive or impossible. A streaming algorithm processes values as they arrive and keeps only the state needed for its result.

Some results can be exact with constant memory: the minimum and maximum of an integer stream do not require remembering earlier values. Other questions, such as the exact number of distinct arbitrary identifiers in a one-pass stream, generally require memory that grows with the number of identifiers. When exact state is too large, a bounded summary can trade precision for a stated error guarantee.

The useful question is not simply “can PHP yield this data lazily?” It is: “what must remain in memory after each item, what information can be discarded, and what guarantee does the final answer provide?”

## Mental Model

```text
source → one item → update compact state → discard or persist item
                        │
                        └── current answer / bounded summary
```

The input may be a file, database cursor, generator, message consumer, or network stream. The algorithm's **state** is the information it retains between items. Its memory bound depends on that state, not on whether the producer uses `yield`.

For an input of length `n`, a one-pass algorithm reads each item once, in stream order, without needing to revisit prior input. Its state may be O(1), O(k) for a configured sample or counter budget, or O(n) if it stores all unique keys. “Streaming” alone does not promise bounded memory.

## Core Concept

A stream is an ordered sequence whose future elements may not be available yet. A data-stream algorithm is designed around limited passes and limited retained state. Common models include:

- **One-pass:** each item is consumed once; the source may be non-rewindable.
- **Multi-pass:** the source can be scanned again, often from an immutable file, snapshot, or indexed table.
- **Online:** the algorithm can emit or update an answer as each item arrives, without waiting for end-of-input.
- **Offline:** the answer is produced after all input is seen, possibly after a second pass or final reduction.
- **Exact:** the returned answer is guaranteed to equal the result for the complete input, under the stated input model.
- **Approximate:** the algorithm states an error bound and, where applicable, a probability that the bound holds.

These terms describe algorithms and sources, not PHP syntax. A generator can expose values on demand, but it does not imply one pass, a bounded source, or a bounded algorithm. A consumer can keep every yielded value in an array. A generator can itself call `fetchAll()` before its first `yield`. Conversely, a direct `while` loop reading a file can be a one-pass bounded-memory computation without using a generator.

Batching is a different technique. [Chapter 88 — Batching](088-batching.md) collects a bounded group to amortize a sink operation, such as one bulk insert. A batch can be part of a streaming pipeline, but it typically retains O(B) records for a batch size B and addresses write efficiency, transaction scope, and retry boundaries. A streaming summary retains only the state needed to answer a query; it may never store a batch of original records.

## What PHP Does

A PHP generator pauses at `yield` and resumes when the caller requests the next value. It saves the generator's execution state, which can make a producer lazy; it does not erase state held in local variables or by the consumer. `iterator_to_array()`, an unbounded cache in a generator wrapper, or a consumer that appends every value can still materialize the entire stream.

The PHP array is a convenient map and list, not a packed fixed-width numeric vector. A million-entry set or frequency map can use far more memory than its logical keys and counters suggest. An iterable database result may also be buffered by its driver or client library, so inspect that source boundary before concluding that the whole pipeline is streaming. [Chapter 74 — Memory Complexity](074-memory-complexity.md) discusses peak live state and collection overhead. If the algorithm has a stated O(k) number of entries, string key length, object payloads, and PHP container overhead still matter.

## Exact Summaries with Bounded State

A minimum/maximum/count summary is exact with constant algorithm state. It reads each value once, keeps three scalars, and can return `null` extrema for an empty stream:

```php
<?php

declare(strict_types=1);

/**
 * @param iterable<int> $values
 * @return array{count: int, min: int|null, max: int|null}
 */
function summarizeIntegers(iterable $values): array
{
    $count = 0;
    $minimum = null;
    $maximum = null;

    foreach ($values as $value) {
        if (!is_int($value)) {
            throw new InvalidArgumentException('Every value must be an integer.');
        }
        if ($count === PHP_INT_MAX) {
            throw new OverflowException('Input count exceeds the integer range.');
        }

        ++$count;
        $minimum = $minimum === null ? $value : min($minimum, $value);
        $maximum = $maximum === null ? $value : max($maximum, $value);
    }

    return ['count' => $count, 'min' => $minimum, 'max' => $maximum];
}
```

Time is O(n); auxiliary space is O(1). The result is exact for the values observed, and the input contract rejects non-integers. Other exact aggregates may need more state: an exact median generally needs information about a large part of the input, while a sum requires an overflow and numeric-precision policy. Exact distinct count over arbitrary identifiers needs a set of seen identifiers or an external exact index; with `d` distinct values, that set uses O(d) entries.

This distinction is useful before reaching for a sketch. If the domain is known and small—such as one of 12 fixed event categories—an array of 12 exact counters may be both exact and bounded. If the domain is arbitrary and can grow without bound, storing one exact counter per key does not have a fixed memory bound.

## Uniform Sampling with a Reservoir

A bounded sample can retain a representative subset when the final input size is not known in advance. Reservoir sampling keeps the first `k` items, then considers each later item for replacement. For the `(i + 1)`th item, with zero-based index `i` after the initial reservoir is full, choose a uniform integer from `0` through `i`; replace that reservoir slot if the chosen value is less than `k`.

The following implementation uses Algorithm R and PHP's `random_int()`, which returns a uniformly selected integer in the inclusive range. It treats stream positions as distinct, even when item values repeat.

```php
<?php

declare(strict_types=1);

/**
 * Keep a uniform sample without replacement by input position.
 *
 * @param iterable<mixed> $items
 * @param null|callable(int, int): int $drawUniform Testable uniform-integer draw; defaults to random_int().
 * @return list<mixed>
 */
function reservoirSample(
    iterable $items,
    int $sampleSize,
    ?callable $drawUniform = null,
): array {
    if ($sampleSize < 0) {
        throw new InvalidArgumentException('Sample size must be nonnegative.');
    }
    if ($sampleSize === 0) {
        return [];
    }

    $drawUniform ??= static fn (int $minimum, int $maximum): int =>
        random_int($minimum, $maximum);

    $reservoir = [];
    $seen = 0;

    foreach ($items as $item) {
        if ($seen === PHP_INT_MAX) {
            throw new OverflowException('Stream length exceeds the integer range.');
        }

        if ($seen < $sampleSize) {
            $reservoir[] = $item;
        } else {
            $slot = $drawUniform(0, $seen);
            if (!is_int($slot) || $slot < 0 || $slot > $seen) {
                throw new UnexpectedValueException('Random draw is outside the requested range.');
            }
            if ($slot < $sampleSize) {
                $reservoir[$slot] = $item;
            }
        }

        ++$seen;
    }

    return $reservoir;
}
```

At every point the reservoir is a uniform sample without replacement of size `min(k, itemsSeen)` from the positions seen so far. When the complete stream has `N` positions and `N >= k`, each size-k subset has probability `1 / binomial(N, k)`. Therefore each individual position has inclusion probability `k/N`. This guarantee depends on drawing independently and uniformly at every step; a custom `$drawUniform` callback is valid only if it provides that contract. Replacing `random_int()` with a biased or predictable source changes the guarantee. The optional callback is useful for deterministic branch tests, but those tests do not prove the statistical property. `random_int()` is designed for cryptographically secure uniform integers, though that level of security is usually more than a report sampler needs and may cost more than a non-cryptographic uniform PRNG.

The algorithm makes O(n) random draws after filling the initial reservoir and retains O(k) items. That is bounded by `k`, not by total stream length. If `k` is too large, or each sampled item is a large hydrated object, the reservoir can still exceed the memory budget. Sampling records is also not the same as sampling distinct values: repeated values at different positions can both appear.

A sample does not make every statistic unbiased or every downstream decision safe. It is appropriate for exploratory analysis or approximate inspection only when the sampling design matches the question. Stratified or weighted populations need a sampling design that accounts for their inclusion probabilities.

## Deterministic Frequent-Item Summary: Misra–Gries

Suppose an event stream has a huge identifier domain, but an operator wants candidates for the most frequent event types. An exact map stores one counter for every distinct identifier and grows with the data. The Misra–Gries algorithm instead keeps at most `m` candidate counters.

For each item, increment its counter if present. If there is an empty slot, assign one to the new item with count one. If all `m` counters are occupied and a new item is absent, decrement every active counter and discard any that reach zero. The update is deterministic; there is no hash-collision probability in its accuracy guarantee.

```php
<?php

declare(strict_types=1);

/**
 * Return at most $capacity frequent-item candidates and their lower-bound counts.
 *
 * @param iterable<string> $items
 * @return array{
 *     itemsProcessed: int,
 *     decrementRounds: int,
 *     candidates: list<array{item: string, estimate: int}>
 * }
 */
function misraGries(iterable $items, int $capacity): array
{
    if ($capacity < 1) {
        throw new InvalidArgumentException('Capacity must be at least one.');
    }

    /** @var array<string, int> prefixed item ID => counter */
    $counters = [];
    $itemsProcessed = 0;
    $decrementRounds = 0;
    $prefix = 'item:';

    foreach ($items as $item) {
        if (!is_string($item)) {
            throw new InvalidArgumentException('Every item must be a string.');
        }
        if ($itemsProcessed === PHP_INT_MAX) {
            throw new OverflowException('Stream length exceeds the integer range.');
        }
        ++$itemsProcessed;

        $key = $prefix . $item;
        if (isset($counters[$key])) {
            ++$counters[$key];
            continue;
        }

        if (count($counters) < $capacity) {
            $counters[$key] = 1;
            continue;
        }

        foreach ($counters as $candidateKey => &$count) {
            --$count;
            if ($count === 0) {
                unset($counters[$candidateKey]);
            }
        }
        unset($count); // Do not keep a reference to the last counter.
        ++$decrementRounds;
    }

    $candidates = [];
    foreach ($counters as $key => $estimate) {
        $candidates[] = [
            'item' => substr($key, strlen($prefix)),
            'estimate' => $estimate,
        ];
    }

    return [
        'itemsProcessed' => $itemsProcessed,
        'decrementRounds' => $decrementRounds,
        'candidates' => $candidates,
    ];
}
```

Let `N` be the number of input items, `m` the number of counters, and `D` the number of decrement rounds. Every decrement round cancels one occurrence from each of the `m` active candidates and the currently arriving item, so `D <= floor(N / (m + 1))`. A candidate's final counter is a lower bound on its true frequency, and the undercount is at most `D`. Any item with true frequency **greater than** `N / (m + 1)` must remain among the candidates; items below that threshold may also remain, so the result can contain false positives. The strict “greater than” matters: an item exactly at the threshold is not guaranteed to survive.

For example, nine counters guarantee that every item occurring more than one tenth of the stream is a candidate, but the candidate's stored count may be below its true count by as many as `floor(N/10)`. The output is not an exact top-nine ranking. It is a bounded candidate set with a deterministic additive-frequency error bound.

This implementation takes O(m) entries and counters. Each non-decrement update does a PHP associative-array lookup, which is expected constant time under ordinary hash-table behavior. A decrement round scans at most `m` entries, but it can happen at most `N/(m+1)` times, so the total scan work is O(N) for this update model; key hashing and string length still add cost. The per-item keys are prefixed so integer-looking identifiers remain distinct from PHP integer array keys. Limit identifier length if stream input is untrusted.

If exact final frequencies are required, a replayable source can be scanned a second time and only candidate IDs recounted. That second pass converts candidates into exact counts for the same snapshot; it does not recover a key that the summary discarded. If the source is a one-shot socket or mutable query with no snapshot, exact verification requires durable staging or a different design.

## Another Approximate Summary: Count-Min Sketch

A Count-Min Sketch estimates point frequencies using a two-dimensional table of counters. Each incoming identifier increments one bucket in every row, chosen by a row-specific hash function. A query takes the minimum of the corresponding counters; collisions can add noise, so for nonnegative frequency updates the estimate is one-sided and may overcount.

For the standard construction, choose `0 < ε < 1` as the error parameter and `0 < δ < 1` as the failure probability, set width `w = ceil(e / ε)` (where `e` is Euler’s number) and depth `d = ceil(ln(1 / δ))` (natural logarithm), and use the hash family required by the construction. For a **fixed queried item**, its estimate `f̂` satisfies `f ≤ f̂ ≤ f + εN` with probability at least `1 - δ`, where `N` is the total number of unit-weight insertions. The table stores `w × d` counters, and one update or one query touches `d` counters. This is an additive, one-sided frequency bound for nonnegative updates—not a relative-error guarantee for every low-frequency item, and not a guarantee that every query in an arbitrarily large set is correct simultaneously.

The mathematics depends on the hash-family assumptions and counter model. Calling PHP's `hash()` function does not by itself prove that a construction has the required independence or failure probability. For a production sketch, use a vetted implementation, document its hash assumptions, parameterize `ε` and `δ`, and test its implementation against exact counts on bounded fixtures. PHP nested arrays of `w × d` counters can use far more memory than the abstract counter count suggests; a packed/native representation or a service designed for sketches may be more appropriate. The original paper gives the construction and point-query bound.

## Errors, State, and Checkpoints

An algorithm's accuracy is not the same as operational correctness. A restart can lose an in-memory summary. Replaying a source can double an output side effect. A persisted summary can be interpreted incorrectly after code changes. For long-running or restartable jobs, define:

- what offset, cursor, partition, or timestamp identifies processed input;
- whether state is committed before or after the source checkpoint;
- whether the sink operation is idempotent under replay;
- the summary algorithm and version, capacity/accuracy parameters, and source snapshot;
- how memory, item size, and input rate are bounded;
- what happens when a stream is malformed, interrupted, late, or unavailable.

A safe checkpoint often needs the algorithm state and source position to advance consistently with sink effects. If the sink succeeds but the checkpoint fails, replay may happen. If the checkpoint advances before the sink is durable, records may be lost. A streaming algorithm alone does not solve these distributed failure modes; [Chapter 88 — Batching](088-batching.md) covers partial progress and transactional batches, while later distributed-systems chapters cover delivery and checkpoint protocols.

## Performance and Memory

Let `n` be the number of input items, `k` a reservoir size, `m` Misra–Gries counters, `d` distinct identifiers, and `w × dSketch` the number of sketch cells where `dSketch` is sketch depth.

| Computation | Time for n items | Retained algorithm state | Accuracy |
| --- | --- | --- | --- |
| Exact integer min, max, count | O(n) | O(1) | Exact within input/integer contract |
| Reservoir sample of size k | O(n) random draws | O(k) sampled values | Uniform without replacement by position, given uniform draws |
| Exact frequency map | Expected O(n) map updates | O(d) keys and counters | Exact |
| Misra–Gries with m counters | O(n) amortized hash/map work | O(m) keys and counters | Candidate guarantee; additive undercount at most `floor(n/(m+1))` |
| Count-Min point-frequency summary | O(n × dSketch) updates | O(w × dSketch) counters | Per-query one-sided additive error `εn` with probability `1-δ` under construction assumptions |

These bounds exclude input payloads and emitted output. A reservoir retaining O(k) full objects can be much larger than O(k) integers. An exact map's O(d) keys may dominate its counters. A packed counter table may have predictable cells but still require native allocation and careful overflow handling. Benchmark on the deployed PHP version and representative identifiers.

For an infinite stream there is no “finished result” unless the algorithm emits periodic snapshots or maintains a continuous answer. Define reset epochs and retention. A summary over “since process start” may become meaningless after a worker restart unless state is checkpointed. Also observe source lag, items processed, dropped or malformed records, summary parameters, checkpoint age, memory use, and sink latency; these measurements reveal whether the stream is being consumed at the intended rate.

## Security and Data Boundaries

Bound input item size and processing cost. In Misra–Gries, the number of stored keys is capped but one attacker-controlled identifier can still be arbitrarily long. In a reservoir, a single retained object may contain a large payload. Validate before updating state, and store compact canonical keys or identifiers where possible.

Approximate summaries can leak information too. A frequency result may reveal that a user, tenant, or secret token appeared even if its count is approximate. Apply access control and retention policies to the summary and to any sampled raw records. Sampling does not anonymize data, and probabilistic output is not a privacy guarantee.

## Testing

Test the input model, state bound, and guarantee:

1. For exact min/max/count, compare with a materialized reference on small arrays; test empty input, singleton input, invalid values, and count-overflow handling.
2. For reservoir sampling, test output size `min(k, n)`, verify selected positions came from the stream, and ensure no input position is selected twice. Use an injected deterministic draw callback to exercise replacement and no-replacement branches; do not assert uniformity from a small number of random runs.
3. For Misra–Gries, compute exact frequencies on bounded test streams. Assert every candidate estimate is at most its true frequency, the undercount is at most the reported decrement count and `floor(n/(m+1))`, and every item above the strict threshold appears among candidates.
4. Test Misra–Gries with capacity one, all-identical data, all-unique data, empty input, numeric-looking string IDs, invalid item types, and repeated cancellation rounds.
5. For a second-pass verifier, use an immutable/replayable fixture and verify exact candidate counts; test that source-version mismatches are rejected by the application boundary.
6. For a Count-Min implementation, compare estimates with exact frequencies across collision-heavy fixtures and confirm the selected parameters match the documented `ε`, `δ`, and hash-family contract. Random trials can find bugs but do not prove a probability guarantee.
7. Check memory growth for increasing stream lengths with fixed parameters. Exact maps should grow with distinct cardinality; bounded summaries should keep their configured number of entries, while payload/key size is measured separately.
8. Simulate interruption around sink and checkpoint boundaries, then prove that replay neither loses data nor duplicates non-idempotent effects.

Keep correctness tests separate from benchmarks. Timing assertions in unit tests are fragile; verify bounded entry counts and algorithm invariants directly, then measure CPU, peak PHP memory, I/O, and checkpoint throughput with production-like input.

## Common Mistakes

- Treating a generator as proof that the complete pipeline has bounded memory.
- Confusing one-pass processing with batching or with asynchronous processing.
- Keeping every unique identifier while claiming constant memory.
- Using an approximate counter without stating absolute/relative error and failure probability.
- Calling a hash function and assuming that it satisfies a sketch's formal hash-family assumptions.
- Interpreting Misra–Gries counters as exact frequencies or as a fully sorted top-k list.
- Claiming an unbiased reservoir sample when the replacement draws are not uniform.
- Testing probability guarantees only with a few observed random outputs.
- Storing large payloads in a bounded number of counters or sample slots and assuming entry count bounds bytes.
- Saving a source offset without saving compatible algorithm state, or advancing either one out of sync with side effects.
- Assuming a local summary is durable, shared across PHP workers, or privacy-preserving.

## Senior Engineer Thinking

Begin with the query and the source. Can the source be replayed? Is one pass required? Does the answer need to be exact? Is the data domain bounded? Which error is acceptable, and is it deterministic or probabilistic? These answers define the memory lower bound and whether a summary algorithm is appropriate.

Then state the guarantee in a sentence that an operator can understand. “Nine counters find all item types appearing more than ten percent of this snapshot, with possible extra candidates” is actionable. “Fast approximate top items” is not. For a randomized sketch, include the error scale, confidence, query scope, and hashing assumptions. For every design, include input ordering, memory per payload, restart state, and replay behavior.

Finally compare a custom stream algorithm with database aggregation, a bounded batch, or an existing stream processor. If the database already has an indexed grouping operation over a fixed snapshot, shipping every row into PHP may be wasteful. If PHP can transform a one-pass file in O(1) state, that may be operationally simpler than adding a distributed service. Choose based on total CPU, memory, network, durability, and maintenance costs.

## Exercises

1. Extend `summarizeIntegers()` to calculate an exact average as a rational pair `(sum, count)`. Define overflow behavior and explain why converting every value to float can lose integer precision.
2. Implement a bounded Misra–Gries summary for a configurable threshold `φ`. Derive the minimum counter capacity that guarantees all items with frequency greater than `φN` remain candidates.
3. Add a second pass that returns exact frequencies for Misra–Gries candidates from the same immutable source snapshot. State what changes if the source is one-shot or mutable.
4. Implement reservoir sampling with a test-only injected uniform-index function. Prove the inclusion probability for an item seen at position `i`, then test structural behavior without relying on a statistical unit test.
5. Suppose a stream has 100 million records but only 20 categories. Compare an exact 20-counter summary, an exact unbounded frequency map, Misra–Gries, and a database `GROUP BY`. State the memory and operational trade-offs.
6. Configure a Count-Min Sketch for a fixed-query additive error target and failure probability. Calculate width, depth, and counter count; explain why the guarantee does not automatically cover every unbounded query simultaneously.
7. Design a checkpointed event summarizer that writes a daily report. Identify how the source cursor, summary state, output write, and restart version remain consistent after a crash.

## Review Questions

1. How is a streaming algorithm different from a PHP generator?
2. How does batching differ from a bounded-state stream summary?
3. Which exact summaries can be computed with constant state, and why does an exact arbitrary-key distinct count need more?
4. What probability guarantee does reservoir sampling provide for each input position?
5. Why is Misra–Gries deterministic, and what does its counter value estimate?
6. With `m` Misra–Gries counters, what frequency threshold guarantees candidate inclusion, and why may there be false positives?
7. What are the one-sided error and confidence bounds for a Count-Min point query under its stated construction?
8. Why does `hash()` alone not establish a Count-Min hash-family guarantee?
9. What state and source position should a restartable summarizer checkpoint together?
10. Why is a fixed item-count bound not always a fixed byte bound in PHP?

## Summary

Streaming algorithms process input in one or a small number of passes while retaining only the state required by the query. Lazy generators control when values are produced but do not guarantee that producers, algorithms, or consumers avoid materializing the stream. Batching groups records to improve sink efficiency and usually retains a bounded batch; it solves a different problem.

Some summaries, such as integer minimum, maximum, and count, are exact with constant state. Reservoir sampling keeps a uniform sample of size `k` with O(k) state. Misra–Gries keeps at most `m` candidates and deterministically guarantees that all items occurring more than `N/(m+1)` times remain candidates, while their counters undercount by at most `floor(N/(m+1))`. Count-Min provides a randomized one-sided additive point-query guarantee only under its specified parameters and hash assumptions. Choose the guarantee, memory bound, input-pass model, restart protocol, and data boundary before adopting a stream summary.

## References

- [PHP Manual: Generators overview](https://www.php.net/manual/en/language.generators.overview.php)
- [PHP Manual: `random_int()`](https://www.php.net/manual/en/function.random-int.php)
- [PHP Manual: `SplQueue`](https://www.php.net/manual/en/class.splqueue.php)
- [Misra and Gries, “Finding repeated elements” (1982)](https://doi.org/10.1016/0167-6423(82)90012-0)
- [Anderson et al., “A High-Performance Algorithm for Identifying Frequent Items in Data Streams” (2017), including the Misra–Gries error lemma](https://conferences.sigcomm.org/imc/2017/papers/imc17-final255.pdf)
- [Cormode and Muthukrishnan, “An Improved Data Stream Summary: The Count-Min Sketch and its Applications” (2005)](https://archive.dimacs.rutgers.edu/~graham/pubs/papers/encalgs-cm.pdf)
- [Vitter, “Random Sampling with a Reservoir” (1985)](https://doi.org/10.1145/3147.3165)
- [Chapter 74 — Memory Complexity](074-memory-complexity.md)
- [Chapter 87 — Sliding Windows](087-sliding-windows.md)
- [Chapter 88 — Batching](088-batching.md)
- [Chapter 89 — Memoization](089-memoization.md)
