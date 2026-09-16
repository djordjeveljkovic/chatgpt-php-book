---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 81
title: Binary Search
slug: binary-search
status: complete
summary: ../../_ai/chapter-summaries/081-binary-search-summary.md
---

# Chapter 81 — Binary Search

## Why This Matters

A product catalog may contain tens of thousands of prices, a worker may load a compact schedule snapshot, or an application may need the first policy that applies after a timestamp. If the values are already sorted, binary search can locate a boundary with a small number of comparisons. It does this by eliminating half of the remaining candidates at each step.

That speed depends on a strict precondition: the searched sequence must already be ordered according to the same rule the search uses. Binary search over an unsorted array is not a slower version of a correct search; it can return a wrong answer while appearing to work on some inputs. The implementation also needs an exact answer to questions such as which duplicate to return, where a missing value belongs, and whether an empty range terminates.

## Mental Model

Binary search maintains a range that may still contain the answer:

~~~text
sorted values + comparison rule + range invariant → boundary or match
~~~

At each iteration it inspects the midpoint. The ordering tells it which half cannot contain the desired boundary, so it discards that half. It does not inspect every value.

For a list of five values, the midpoint is not a guess about where the answer “probably” is. It is a probe selected so that, whatever the comparison says, a large part of the remaining range can be ruled out. With **n** values, the number of probes grows in proportion to **log₂(n)**, rather than **n**.

The algorithm finds a position in an ordered sequence. It does not make that sequence ordered, keep it fresh, define duplicate policy, or prove that a database record is still current.

## Core Concept

The most useful reusable form of binary search finds a boundary. The lower bound is the first position whose value is greater than or equal to a target. The upper bound is the first position whose value is strictly greater than the target.

For sorted values:

~~~text
values:     [1, 2, 2, 2, 5]
lowerBound(2) = 1
upperBound(2) = 4

matching values occupy [1, 4)
~~~

The interval notation **[1, 4)** includes position 1 and excludes position 4. The number of matching values is **upperBound - lowerBound**, here three. If both bounds are equal, there is no equal value. If only one matching position is needed, lower bound plus one comparison provides the first match.

This boundary-first definition handles duplicates explicitly. A search that returns an arbitrary equal element often hides whether the caller wanted the first occurrence, the last occurrence, or all occurrences.

## How It Works

### State the sorted-input precondition

Binary search assumes the sequence is sorted under a consistent ordering. If searching integers in ascending order, the precondition is:

~~~text
for every valid i: values[i] <= values[i + 1]
~~~

The sequence must also support the indexing used by the algorithm. A PHP array is an ordered map; its values are not guaranteed to have consecutive integer keys. For direct indexing by positions from zero to **count($values) - 1**, this chapter's implementations require a list.

You can check the list shape with **array_is_list()**, available since PHP 8.1, when data first enters a trusted in-memory representation. Doing that check for every query would inspect the keys each time and can cost O(n), erasing the O(log n) search bound. A PHPDoc **list<int>** annotation documents a contract to a reader and static analyzer; it does not validate a value at runtime.

The order itself is a semantic precondition too. If a list was sorted case-insensitively but is searched using case-sensitive comparison, or sorted by one record field but searched by another, the discarded half may contain the desired item.

### Use a half-open interval

A half-open range includes its low endpoint and excludes its high endpoint:

~~~text
[low, high)
~~~

For a list of length **n**, begin with **low = 0** and **high = n**. This represents every valid position, including the empty list. No midpoint is valid when **low === high**, so the loop condition is simply **low < high**.

For lower bound, keep this invariant:

~~~text
Every position before low has value < target.
Every position at or after high has value >= target.
~~~

The still-unknown answer is inside **[low, high)**. At the midpoint:

- If its value is less than the target, that position and everything before it are too small. Move **low** to **middle + 1**.
- Otherwise, the midpoint may itself be the first acceptable position. Move **high** to **middle**.

When the range is empty, **low === high**. The invariant then says that every earlier value is smaller and every value from that position onward is at least the target. That position is exactly the lower bound.

An implementation of the lower bound for integers is:

~~~php
<?php

declare(strict_types=1);

/**
 * Return the first position whose value is greater than or equal to $target.
 *
 * @param list<int> $values Values sorted in ascending numeric order.
 */
function lowerBoundInts(array $values, int $target): int
{
    $low = 0;
    $high = count($values);

    while ($low < $high) {
        $middle = $low + intdiv($high - $low, 2);

        if ($values[$middle] < $target) {
            $low = $middle + 1;
        } else {
            $high = $middle;
        }
    }

    return $low;
}
~~~

The function returns a position from **0** through **n**. A return value of **n** means every value is less than the target; it is a valid insertion boundary, but not a valid position to index. The empty list returns zero.

### Find the other edge of duplicates

Upper bound follows the same interval shape, but asks for the first value strictly greater than the target. Its invariant is:

~~~text
Every position before low has value <= target.
Every position at or after high has value > target.
~~~

When the midpoint value is less than or equal to the target, the boundary must be to its right. Otherwise it is at or to its left.

~~~php
/**
 * Return the first position whose value is greater than $target.
 *
 * @param list<int> $values Values sorted in ascending numeric order.
 */
function upperBoundInts(array $values, int $target): int
{
    $low = 0;
    $high = count($values);

    while ($low < $high) {
        $middle = $low + intdiv($high - $low, 2);

        if ($values[$middle] <= $target) {
            $low = $middle + 1;
        } else {
            $high = $middle;
        }
    }

    return $low;
}
~~~

Both functions have the same loop shape. The one comparison difference encodes whether equal values belong to the discarded prefix. Naming the two functions by the boundary they find is safer than maintaining separate, vaguely named “find” implementations whose duplicate behavior is unclear.

To test for a value and get the first matching position, first find the lower bound, then check that the result is less than the list length and equal to the target. To get all equal values, use both bounds. To insert a new value while placing it after existing equals, use upper bound; to place it before equals, use lower bound.

Finding a position is logarithmic, but inserting into a PHP list at that position generally requires shifting later elements and costs O(n). The search does not make the update logarithmic.

### Avoid off-by-one and nontermination errors

The high endpoint is **n**, not **n - 1**, because it is exclusive. That choice gives one implementation for empty and non-empty lists. It also means code must check a returned position before indexing it.

The midpoint belongs to the current interval: **low <= middle < high** whenever **low < high**. If the midpoint is too small for lower bound, advance to **middle + 1**. If it is already a candidate, keep it in the range by setting **high = middle**. Setting **high = middle - 1** would discard a possible answer. Failing to advance **low** in the first case can repeat the same midpoint forever.

A common alternative is an inclusive interval ending at **n - 1**. That convention can work, but empty lists and updates around the endpoints need extra cases. Do not mix the update rules from inclusive and half-open formulations.

### Calculate the midpoint safely

A familiar midpoint expression is:

~~~php
$middle = intdiv($low + $high, 2);
~~~

For sufficiently large indices, adding **low + high** could overflow before division. Use:

~~~php
$middle = $low + intdiv($high - $low, 2);
~~~

Here **high >= low >= 0**, so **high - low** cannot overflow. Adding at most half that difference to **low** produces a result no greater than **high**. Real PHP arrays are constrained by addressable memory long before this theoretical limit matters on ordinary machines, but the safer formula is just as clear and remains correct across integer sizes.

### Generalize with one comparison contract

For records or domain-specific values, accept a comparator. It must return a negative integer when the item sorts before the target, zero when it compares equal, and a positive integer when it sorts after the target. It must be consistent and transitive, and it must agree with the comparison used to sort the list.

~~~php
/**
 * @template T
 * @template U
 * @param list<T> $values
 * @param U $target
 * @param callable(T, U): int $compare
 */
function lowerBound(
    array $values,
    mixed $target,
    callable $compare,
): int {
    $low = 0;
    $high = count($values);

    while ($low < $high) {
        $middle = $low + intdiv($high - $low, 2);

        if ($compare($values[$middle], $target) < 0) {
            $low = $middle + 1;
        } else {
            $high = $middle;
        }
    }

    return $low;
}

/**
 * @template T
 * @template U
 * @param list<T> $values
 * @param U $target
 * @param callable(T, U): int $compare
 */
function upperBound(
    array $values,
    mixed $target,
    callable $compare,
): int {
    $low = 0;
    $high = count($values);

    while ($low < $high) {
        $middle = $low + intdiv($high - $low, 2);

        if ($compare($values[$middle], $target) <= 0) {
            $low = $middle + 1;
        } else {
            $high = $middle;
        }
    }

    return $low;
}
~~~

The comparator's zero result defines search equality. It might mean equal integer IDs, equal normalized email addresses, or equal sort keys; it does not automatically mean that two records are identical in every field. If sorting treats two records as equal while the search comparator distinguishes them, the list may not be partitioned as the search expects. If the comparator is inconsistent—for example, it reads a mutable property or applies a different normalization on different calls—the search has no reliable result.

Chapter 79 discusses comparator laws and the PHP sorting functions. PHP's sorting comparator contract also uses negative, zero, and positive results; its documentation notes that non-integer results are cast to integers. A binary-search comparator should likewise return an integer sign and should avoid subtraction-based comparisons that can overflow or lose precision.

## What PHP Does

The algorithm is implemented in userland PHP and uses integer variables for the bounds and midpoint. Array access retrieves the value at the selected integer key. PHP's arrays are ordered maps, as described in Chapter 75 and the PHP Manual; the list precondition is what makes a positional index correspond to the intended value.

The built-in **array_is_list()** can check consecutive keys, but it does not check that values are sorted. Sorting validation needs domain-specific comparisons and is itself a traversal. Usually an application establishes the shape and ordering once—when loading, normalizing, or constructing a snapshot—and then reuses that trusted representation for queries.

The Manual specifies PHP's observable array and comparison behavior, but does not promise a formal complexity bound for array access or a particular cost for every runtime configuration. O(log n) here counts comparator calls in the algorithmic model. Comparator work, array representation, allocations elsewhere in the request, and CPU behavior still affect wall-clock time.

## Practical Example: Select a Shipping Tier

An application may load a small, immutable carrier configuration at worker startup. Each tier contains its maximum supported package weight and the fee for that range. The tier list is sorted by maximum weight and has unique, strictly increasing limits:

~~~php
<?php

declare(strict_types=1);

/**
 * @param list<array{max_grams: int, price_cents: int}> $tiers
 */
function compareTierLimitToWeight(array $tier, int $weight): int
{
    return $tier['max_grams'] <=> $weight;
}

/**
 * @param list<array{max_grams: int, price_cents: int}> $tiers
 */
function shippingPriceCents(array $tiers, int $packageWeightGrams): ?int
{
    if ($packageWeightGrams < 0) {
        throw new InvalidArgumentException('Package weight cannot be negative.');
    }

    $position = lowerBound(
        $tiers,
        $packageWeightGrams,
        compareTierLimitToWeight(...),
    );

    if ($position === count($tiers)) {
        return null; // No configured tier accepts this package.
    }

    return $tiers[$position]['price_cents'];
}

$tiers = [
    ['max_grams' => 500, 'price_cents' => 450],
    ['max_grams' => 1_000, 'price_cents' => 650],
    ['max_grams' => 2_000, 'price_cents' => 900],
    ['max_grams' => 5_000, 'price_cents' => 1_400],
];

$fee = shippingPriceCents($tiers, 1_250);
// 900: the first tier whose maximum is at least 1,250 grams.
~~~

The named comparator projects each tier onto the key used for ordering. PHP 8.1 first-class callable syntax, shown with **compareTierLimitToWeight(...)**, creates a callable for the generic boundary function; an ordinary closure can be passed instead on codebases that prefer it.

The configured limits must be ordered, and configuration loading should reject malformed or duplicate limits according to the business rule. The tier array must remain a list and must not be mutated out of order after validation. If carrier prices are stored in a database and can change independently, query that source with the appropriate predicate and transaction/freshness guarantees rather than trusting a worker's old snapshot.

## Performance and Memory

For a sorted list of **n** values, each iteration reduces the candidate interval to at most about half its previous size. The number of comparisons is O(log n) in the worst case. The iterative implementations keep only a few integer variables, so auxiliary space is O(1). A recursive version also needs a call stack proportional to O(log n) and has no advantage here.

Those bounds describe search after ordering already exists. Sorting an unsorted list costs O(n log n) comparisons with PHP's built-in comparison sorting model; see Chapter 79 for the API and implementation caveats. For **q** searches, a rough comparison is:

~~~text
one linear scan:             O(n)
q linear scans:               O(qn)
sort once, then search:       O(n log n + q log n)
~~~

That comparison leaves out sorting memory, snapshot construction, updates, comparator expense, and the cost of preserving a second representation. If the data changes often, restoring sorted order can dominate. If a search occurs only once, a linear scan may be faster for a small collection and is simpler when no ordering exists. Benchmark the actual workload instead of treating asymptotic notation as a timing promise.

The iterative algorithm itself allocates no result collection and makes O(log n) probes. A PHP list of records can still occupy much more memory than a packed native integer vector, as discussed in Chapters 49, 50, and 74. Sorting a large array may require additional working memory depending on the PHP version and implementation. Measure peak memory alongside CPU time for realistic data sizes.

## When Linear Search or a Database Index Is Better

Use a linear scan when the collection is small, used for one query, not sorted, or exposed only as a stream. A scan can also be clearer for predicates that do not define a single monotonic boundary. Binary search cannot efficiently find “the first record that is available” in a list sorted by start time if availability fluctuates arbitrarily; those values do not form one true-then-false partition.

For persistent records, a database index often belongs closer to the data than a PHP copy. A query such as “first shipping rule with maximum weight at least this package's weight” can use an indexed ordered column, subject to the database's actual query plan and schema. The index does not guarantee constant work: selectivity, data distribution, joins, ordering, and rows returned matter. Inspect plans for important queries and keep the database's collation and comparison behavior aligned with the application rule.

Loading every row into PHP to binary-search it can add network transfer, hydration, memory, snapshot freshness, and worker synchronization costs. A local binary search is appropriate when the application already owns a bounded, stable, ordered in-memory snapshot or when multiple ordered operations use that representation. It does not replace transaction isolation or a database constraint for a concurrent write invariant.

## Testing

Test boundaries rather than just a successful midpoint match. A set of hand-picked cases should include an empty list; a target smaller than every value; larger than every value; below, equal to, and above interior values; a one-element list; and repeated values at the beginning, middle, and end.

For **[1, 2, 2, 2, 5]**, the target **2** must produce lower bound 1 and upper bound 4. The target **3** produces lower bound and upper bound 4, an empty match range. The target **0** produces both bounds at 0, and the target **9** produces both at 5. These exercise the sentinels at both ends as well as duplicate behavior.

A simple test can verify the known duplicate case:

~~~php
$values = [1, 2, 2, 2, 5];

if (lowerBoundInts($values, 2) !== 1) {
    throw new LogicException('Incorrect lower bound for duplicate values.');
}

if (upperBoundInts($values, 2) !== 4) {
    throw new LogicException('Incorrect upper bound for duplicate values.');
}
~~~

For stronger confidence, generate many small sorted lists and compare the bounds to a straightforward reference scan. Also check the invariant directly: every value before lower bound is less than the target; every value from lower bound onward is at least it; every value before upper bound is at most the target; and every value from upper bound onward is greater. This property-based reasoning catches off-by-one errors without relying on timing.

For comparator-based searches, sort with the same ordering rule used by the search. Include ties and confirm the intended equivalence class. Reject or normalize malformed input before comparison. Do not test performance by asserting that a test finishes in a particular number of milliseconds; measure performance separately with representative data and repeatable tools.

## Common Mistakes

- Searching a list that is not sorted under the query's comparison rule.
- Applying positional indexing to an associative or sparse PHP array.
- Assuming lower bound returns an exact match; it returns an insertion boundary.
- Forgetting to check a returned position before indexing, especially when it equals **count($values)**.
- Returning an arbitrary duplicate when the caller needs the first, last, or full range.
- Mixing inclusive-range and half-open-range update rules.
- Setting **high = middle - 1** in the half-open lower-bound algorithm and dropping a possible answer.
- Failing to advance **low** past a midpoint known to be too small, causing nontermination.
- Using different normalization or comparator rules for sorting and searching.
- Sorting on every call, then claiming each lookup costs only O(log n).
- Expecting binary search to make insertion into a PHP list logarithmic.
- Copying database-owned data into a PHP array when an indexed query is the correct ownership boundary.

## Senior Engineer Thinking

Before adopting binary search, verify the data contract: who established the order, whether the order survives updates, what constitutes equality, and what the boundary means to callers. Then account for the cost of creating and refreshing that representation. A few microseconds saved by lookup are irrelevant if loading, sorting, or coordinating the snapshot costs more or makes the result stale.

The strongest APIs expose the useful boundary directly. “First tier with capacity at least this weight” is a clearer contract than “find the tier.” For duplicate values, returning a half-open range makes the policy explicit and supports counts and range queries without rescanning. For a database-owned collection, ask whether SQL can express the same boundary using an index and preserve the needed consistency.

## Exercises

1. Implement a function that returns the first and one-past-last positions of a target in an ascending list, or an empty range if absent. State the loop invariant for each bound.
2. Given ascending event timestamps, find the first event at or after a cutoff and explain how this differs from finding the latest event at or before it.
3. Adapt the comparator-based search to case-insensitive usernames. Define normalization, sorting, equality, and duplicate behavior; then ensure the comparator is consistent with all four.
4. Compare a one-off scan, sorting followed by repeated binary searches, and a database query for a collection that grows over time. Include build, refresh, network, and memory costs.
5. Design tests that expose each termination bug caused by an incorrect midpoint or endpoint update.

## Review Questions

1. What sorted-input and array-shape assumptions does binary search require?
2. State the lower-bound invariant over the half-open interval **[low, high)**.
3. Why does the algorithm return **n** when all values are smaller than the target, and why must callers check before indexing?
4. How do lower and upper bounds specify duplicate behavior?
5. Why is **low + intdiv(high - low, 2)** safer than adding the endpoints first?
6. Which part of the ordering contract defines equality for a comparator-based search?
7. Why can a linear scan be preferable for one search over a small unsorted list?
8. When should a database index own the search, and what costs remain even then?

## Summary

Binary search shrinks a candidate interval by half, giving O(log n) comparisons and O(1) auxiliary space after a sorted representation exists. The sequence must be a positional list ordered under the same consistent comparison rule used by the search. A half-open interval and explicit invariant make termination and endpoint handling easier to prove. Lower bound returns the first value at least as large as the target; upper bound returns the first value larger, so duplicates occupy a clear half-open range. The algorithm does not pay for sorting, updates, memory, or freshness. Use a scan for small or one-off searches, and let an indexed database search persistent data when the database owns it.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: array_is_list()](https://www.php.net/manual/en/function.array-is-list.php)
- [PHP Manual: usort() and its comparison contract](https://www.php.net/manual/en/function.usort.php)
- [PHP Manual: Sorting arrays](https://www.php.net/manual/en/array.sorting.php)
- [Chapter 73 — Big O](./073-big-o.md)
- [Chapter 74 — Memory Complexity](./074-memory-complexity.md)
- [Chapter 75 — Arrays and Hash Maps](./075-arrays-and-hash-maps.md)
- [Chapter 79 — Sorting](./079-sorting.md)
- [Chapter 80 — Searching](./080-searching.md)
- [Chapter 82 — Trees](./082-trees.md)
