---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 173
title: Property-Based Testing
slug: property-based-testing
status: complete
summary: ../../_ai/chapter-summaries/173-property-based-testing-summary.md
---

# Chapter 173 — Property-Based Testing

## Why This Matters

Example-based tests choose a few named inputs. Property-based tests describe an invariant and generate many inputs, including boundary values that a person may not think to write. They are useful for parsers, normalizers, serializers, collections, state transitions, and algorithms where the number of valid combinations is large.

A property is a statement that should hold for every input in a defined domain. The generator is part of the test design: a property over arbitrary bytes says little if the production method accepts only UTF-8 identifiers, while a generator that omits empty strings can hide a real boundary defect.

## Properties and Oracles

A useful property has a clear oracle:

- Normalizing an already normalized value produces the same value.
- Encoding and then decoding a supported value preserves its meaning.
- Sorting a list preserves its elements and produces a nondecreasing order.
- Splitting an interval and joining the pieces preserves the covered range.
- Applying the same idempotent command twice produces the same domain state.

Avoid properties that simply call the implementation twice and compare the results; a shared bug can satisfy both sides. Derive an expected result from a simpler model, an invariant, or an independent representation.

## A Deterministic Generator

PHPUnit does not provide a general property-based engine in its core. A library such as Eris can integrate generators and shrinking with PHPUnit. A small deterministic loop can demonstrate the design and is sometimes sufficient for a focused property.

~~~php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

final readonly class DateRange
{
    public function __construct(public int $start, public int $end)
    {
        if ($start > $end) {
            throw new InvalidArgumentException('Start must not exceed end');
        }
    }

    public function length(): int
    {
        return $this->end - $this->start;
    }
}

final class DateRangePropertyTest extends TestCase
{
    public function testLengthIsNeverNegativeForGeneratedRanges(): void
    {
        mt_srand(173);
        for ($i = 0; $i < 1_000; ++$i) {
            $start = mt_rand(-1_000_000, 1_000_000);
            $end = mt_rand($start, 1_000_000);

            self::assertGreaterThanOrEqual(0, (new DateRange($start, $end))->length());
        }
    }
}
~~~

A fixed seed makes a local failure reproducible, but it is not a substitute for good generation. Record the seed when a property fails and rerun that case as a regression test. Do not use a global random generator whose state other tests can alter; inject a generator or keep the property loop's state local.

The example's generator chooses a valid range. Add separate properties for rejected inputs, such as start greater than end, because invalid-domain behavior is a contract too.

## Shrinking and Minimal Failures

A generator can produce a large failing structure that is difficult to understand. Shrinking repeatedly simplifies it while preserving the failure: a long string may become a one-character string, and a list may become two elements. The smallest counterexample often explains the bug better than the original random input.

When using a library, preserve the failing seed and generated value in the test report. If writing a small generator, provide a shrink function for each structure and ensure the shrinker terminates. A shrinker that changes the input into a valid case can hide the failure; a shrinker that never reaches a minimal case slows diagnosis.

## Generators and Domain Boundaries

Generate values according to the domain:

- identifiers: empty, minimum and maximum length, Unicode, separators, and reserved words;
- money: zero, minimum unit, large values, currencies, and rounding boundaries;
- dates: leap days, time-zone transitions, inclusive and exclusive endpoints;
- JSON: missing fields, explicit null, extra fields, nested arrays, duplicate semantic values;
- permissions: tenant mismatch, revoked membership, resource state transitions.

Use weighted generation when rare but important cases need more coverage. Keep generated input within resource limits so a test cannot allocate gigabytes or create an unbounded request. A property test should be fast enough to run routinely, with a separate stress profile for larger ranges.

## Metamorphic Properties

Some functions have no convenient expected output, but a related input should produce a predictable relation. These are metamorphic properties:

- Reordering independent input records should not change an aggregate total.
- Adding a duplicate record should change a deduplicating operation by zero.
- Encoding an output and decoding it should preserve the selected fields.
- Applying a migration twice should leave the schema in the same state if the migration is designed to be repeatable.

State the relation explicitly and ensure the transformed input stays within the domain. A metamorphic test is not proof of correctness by itself; combine it with examples and domain invariants.

## Stateful and Model-Based Tests

For a stateful component, generate command sequences and compare the system with a small model. A queue model can apply enqueue and dequeue operations to a PHP array, then compare the implementation's visible behavior. Include invalid commands and reset operations. Limit sequence length for quick CI and run longer sequences in a scheduled job.

When a failure is found, preserve the command sequence as a regression test. The sequence is more useful than only the final state because it documents the transition that caused the defect. Ensure each command has a precondition and a defined expected failure; otherwise the model and system may diverge for an accidental reason.

## Integration and Side Effects

Property-based tests work best for pure functions and deterministic domain services. For a database or external API, generate bounded commands and run them against an isolated fixture with cleanup. Do not generate real payments, emails, or unbounded uploads. A property can assert that a retry does not duplicate a row, but the database constraint and transaction behavior still need an integration test against the real engine.

Use generated values in logs only under a test retention policy. A failing input may contain credentials or personal data if the generator is fed production-like fixtures; test data should be synthetic.

## Common Mistakes

- Calling random examples a property without stating an invariant.
- Generating only valid happy-path values.
- Using an uncontrolled random seed that cannot reproduce a failure.
- Omitting shrinking or failing to preserve the counterexample.
- Reusing the implementation as its own oracle.
- Generating inputs too large for normal CI resources.
- Treating a passing property as a replacement for targeted security cases.

## Senior Engineer Thinking

Property-based testing trades a few hand-picked examples for a precise invariant and a distribution of inputs. Invest in generators that represent the domain, shrinkers that explain failures, and oracles independent enough to catch shared bugs. Use it where input combinations are rich, then turn every useful counterexample into a named regression test.

## Exercises

1. Define properties for a money rounding function, including currencies with different minor units.
2. Implement a generator and shrinker for non-overlapping time intervals.
3. Create a model-based test for a bounded queue with invalid dequeue operations.
4. Run a property with a recorded seed, then convert its smallest counterexample into an example-based regression test.

## Review Questions

1. What makes a property different from a collection of random examples?
2. Why are generator boundaries part of the test contract?
3. What does shrinking contribute to debugging?
4. When is a metamorphic property useful?
5. Why should generated tests still be paired with targeted security and integration tests?

## Summary

Property-based testing checks invariants over generated domain inputs. Define an independent oracle, generate valid and invalid boundaries, preserve seeds, shrink failures, and bound resource use. Apply it to pure logic and modelable state transitions, then promote valuable counterexamples to regression tests and keep real integration tests for database and external side effects.

## References

- [PHPUnit documentation](https://docs.phpunit.de/)
- [Eris property-based testing for PHP](https://github.com/giorgiosironi/eris)
- [QuickCheck paper and ecosystem](https://hackage.haskell.org/package/QuickCheck)
- [Martin Fowler: Property-Based Testing](https://martinfowler.com/articles/property-based-testing.html)

