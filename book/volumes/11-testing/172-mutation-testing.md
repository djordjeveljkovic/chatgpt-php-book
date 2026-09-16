---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 172
title: Mutation Testing
slug: mutation-testing
status: complete
summary: ../../_ai/chapter-summaries/172-mutation-testing-summary.md
---

# Chapter 172 — Mutation Testing

## Why This Matters

Coverage can show that a test executed a line without showing whether the test would notice a defect in that line. Mutation testing creates small, deliberate changes to production code and runs the tests. A mutant is killed when a test fails because of the change; it survives when the suite cannot distinguish the mutant from the original.

Mutation testing asks a sharper question than “did this line run?”: “would the suite detect this plausible mistake?” It is a diagnostic tool for test quality, especially around conditions, boundaries, authorization, and state transitions.

## Mutation Operators

A mutation tool applies operators such as:

* replacing greater-than with greater-than-or-equal;
* changing a boolean condition;
* removing a method call or return value;
* replacing a constant;
* changing arithmetic or string operations;
* altering a comparison or negating a condition.

For a quantity rule 1 through 10, changing the upper comparison to a strict less-than should be killed by a test for 10. If it survives, the suite has no evidence for the upper boundary, even if line coverage is 100 percent.

~~~php
<?php

declare(strict_types=1);

function acceptsQuantity(int $quantity): bool
{
    return $quantity >= 1 && $quantity <= 10;
}
~~~

A useful test names both boundaries and rejection:

~~~php
<?php

declare(strict_types=1);

function testQuantityBoundaries(): void
{
    assert(acceptsQuantity(1));
    assert(acceptsQuantity(10));
    assert(!acceptsQuantity(0));
    assert(!acceptsQuantity(11));
}
~~~

The test kills mutations of either comparison. Do not add assertions solely to satisfy a mutant; add the case because the domain contract requires it.

## Infection for PHP

Infection is a mutation-testing framework for PHP that integrates with PHPUnit and other supported test runners. A typical project configures source directories and runs Infection after the ordinary suite and coverage checks:

~~~json
{
    "timeout": 10,
    "source": {
        "directories": ["src"]
    },
    "logs": {
        "text": "build/infection.log",
        "summary": "build/infection-summary.log"
    }
}
~~~

A representative command is infection --test-framework=phpunit. Exact options and configuration keys depend on the installed Infection version; consult its documentation and keep the configuration in the project. Infection can use coverage to limit mutations to executable code, then runs relevant tests for each mutant.

Mutation testing can be expensive. Run a focused set for changed or high-risk code on pull requests and a broader run on a schedule. Cache or reuse analysis only when it does not make results stale or hide changed tests.

## Reading the Result

A mutation report commonly distinguishes killed, escaped or survived, timed out, and ignored mutants. A score is a useful trend, not a universal quality target. Investigate survivors:

1. Read the changed code and the mutation.
2. Ask whether the mutant represents a possible defect.
3. If it does, add a behavior-focused test or strengthen an assertion.
4. If it does not, document or configure a narrow exclusion.
5. Re-run the relevant mutation and confirm the result.

An equivalent mutant produces behavior indistinguishable from the original under the real program semantics. It may be safe to ignore, but an exclusion should be narrow and explained. A timeout may indicate a real performance problem, an infinite loop under the mutant, or an environment limit; classify it before ignoring.

## Security and Domain Mutations

Prioritize mutations in authorization, tenant scoping, validation, payment state, idempotency, and error handling. A mutation that changes an allow decision to deny may be visible in a happy-path test but a missing denial test can let an unsafe branch survive. Mutation testing is especially useful for conditions that combine several booleans, where a line report hides missing combinations.

For side effects, mutate the call or condition that controls publishing, charging, deleting, or granting access. The test should check both the desired effect and its absence on rejected input. A spy can prove an event was sent in a unit test; an integration or feature test must prove the durable boundary also behaves correctly.

## Failure and Threat Analysis

* **Low score treated as failure:** a score becomes a target and teams add shallow assertions. Review survivors by risk.
* **Equivalent mutants:** impossible or semantically identical changes waste time. Exclude narrowly and document why.
* **Slow runs:** too many mutants block delivery. Scope by changed and critical code, and run broad jobs on a schedule.
* **Missing coverage:** unexecuted code is not meaningfully mutation-tested. Fix test selection or source configuration first.
* **Weak assertions:** tests execute but do not distinguish behavior. Add strict contract assertions.
* **Ignored timeouts:** a mutant reveals an unbounded loop or pathological input. Investigate before excluding.
* **Generated code noise:** mutations in vendor or generated files obscure application risk. Configure source directories.

## Workflow and CI

Run the normal suite before mutation testing so ordinary failures are not confused with mutant failures. Keep the baseline test command reproducible and record the PHP, dependency, mutation tool, and test-runner versions. Publish a report that helps reviewers find survivors without exposing secrets or production data.

Do not require every mutant to be killed immediately in a legacy system. Set a baseline, prevent new survivors in high-risk changed code where feasible, and pay down existing survivors with tracked work. A mutation score can fall after a refactor even when tests improve; review the actual survivors and source changes.

## Exercises

1. Write boundary tests for an amount validator and run mutations changing equality, comparison, and rounding.
2. Configure Infection for a small PHP project and classify five surviving mutants as missing test, equivalent, timeout, or out of scope.
3. Choose an authorization policy and mutate its allow condition. Add tests for anonymous, owner, wrong-tenant, and privileged actors.
4. Compare mutation runtime for all source and changed source. Define a CI schedule that preserves useful feedback.

## Review Questions

1. What does a killed mutant demonstrate?
2. Why can a mutation score not replace test design?
3. Which boundary test kills a change from less-than-or-equal to less-than?
4. How should equivalent mutants and timeouts be handled?
5. Why should mutation testing run after the ordinary suite passes?
6. Which security branches deserve mutation priority?

## Summary

Mutation testing evaluates whether tests detect plausible code changes. Use Infection with PHPUnit, read survivors as evidence about missing behavior, prioritize security and domain conditions, distinguish equivalent mutants from weak tests, and control runtime through focused and scheduled runs. Treat the score as a diagnostic trend supported by good test design and coverage configuration.

## References

- [Infection PHP](https://infection.github.io/)
- [Infection Configuration](https://infection.github.io/guide/configuration.html)
- [PHPUnit Documentation](https://docs.phpunit.de/)
- [Martin Fowler: Test Coverage](https://martinfowler.com/bliki/TestCoverage.html)

