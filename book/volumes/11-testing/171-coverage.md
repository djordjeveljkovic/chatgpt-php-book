---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 171
title: Coverage
slug: coverage
status: complete
summary: ../../_ai/chapter-summaries/171-coverage-summary.md
---

# Chapter 171 — Coverage

## Why This Matters

Coverage measures which parts of a program a test run executes. It is useful evidence about untested code, but it is not a correctness score. A test can execute a line and assert the wrong result; another test can miss a critical branch while reporting a high percentage.

Use coverage to find risk and guide questions: which authorization branch has no test, which exception path is unobserved, and which generated adapter is never exercised? Set a threshold as a safety floor after meaningful tests exist, not as the definition of quality.

## What Coverage Measures

Common metrics answer different questions:

* **Line coverage:** which executable lines ran.
* **Statement coverage:** which statements ran, depending on the tool's model.
* **Function or method coverage:** which functions were entered.
* **Branch coverage:** which decision outcomes ran.
* **Path coverage:** which complete combinations of branches ran; this grows quickly and is rarely exhaustive.
* **Condition coverage:** which boolean subconditions were true and false, when the tool supports it.

A report may say 90 percent line coverage while a security check's false branch is never exercised. Branch and condition views often reveal more useful gaps, but no metric understands whether an assertion checks the correct business rule.

## Generate a Report with PHPUnit

PHPUnit can collect coverage through a driver such as PCOV or Xdebug. A project normally configures the source include paths and runs coverage in CI:

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/11.0/phpunit.xsd">
    <source>
        <include>
            <directory>src</directory>
        </include>
    </source>
    <testsuites>
        <testsuite name="unit">
            <directory>tests/Unit</directory>
        </testsuite>
    </testsuites>
</phpunit>
~~~

A representative command is phpunit --coverage-text or phpunit --coverage-html build/coverage, depending on the project and installed driver. Keep generated reports out of version control unless the project publishes them deliberately. Confirm that the report includes the intended source and excludes generated code, vendor code, migrations that are not application behavior, and test fixtures according to the measurement goal.

Coverage collection changes execution cost. Run a fast test suite on every change and a full branch or HTML report in CI or a scheduled job when the overhead is significant. The exact command and XML schema depend on the supported PHPUnit version; use that version's documentation.

## Thresholds and CI

A threshold can prevent a new feature from shipping with an obvious testing gap. It should be enforced against changed or high-risk code where possible, and it should be accompanied by review. Raising a global threshold can encourage meaningless tests, deleting code, or excluding difficult paths.

Treat a falling trend as a question, not an automatic verdict. A legitimate refactor can remove code and change percentages. A new security-sensitive endpoint may deserve tests even if the global number rises. Record why important exclusions exist and review them when the architecture changes.

## Finding Valuable Gaps

Read the source and report together. Focus first on:

* authorization denials, tenant scoping, and state transitions;
* validation failures, exception mapping, and rollback paths;
* retries, timeouts, duplicate delivery, and idempotency;
* boundary values, empty data, concurrency, and resource limits;
* serializers, migrations, adapters, and configuration branches.

A line that is impossible or intentionally defensive can be excluded with a documented reason. A line that is difficult because it calls a provider or process may need an integration or feature test, not an exclusion.

## Coverage and Assertions

Coverage tells you a line ran, not that it was observed correctly. This test can execute a branch but prove little:

~~~php
<?php

declare(strict_types=1);

function accepts(int $value): bool
{
    return $value > 0 && $value <= 10;
}

function weakTest(): void
{
    accepts(7);
}
~~~

A useful test asserts the contract and boundaries:

~~~php
<?php

declare(strict_types=1);

function testAcceptsBoundaries(): void
{
    assert(accepts(1));
    assert(accepts(10));
    assert(!accepts(0));
    assert(!accepts(11));
}
~~~

Coverage and assertion quality complement each other. Mutation testing can reveal tests that execute code without detecting a changed result.

## Exclusions and Generated Code

Exclude code only when it is genuinely outside the behavior being measured or impossible to exercise safely, such as generated proxies or a platform-specific defensive branch. Do not exclude a branch merely because writing its test is inconvenient. Keep exclusions narrow, visible in configuration, and explained in code review.

Separate application source, vendor code, fixtures, and generated artifacts in the coverage configuration. Test framework bootstrap code can be covered by the framework's own tests rather than inflating the application's number.

## Failure and Threat Analysis

* **High line score, weak assertions:** lines execute but outcomes are not checked. Review assertions and use mutation testing.
* **Global threshold gaming:** developers add trivial tests or exclude difficult code. Measure meaningful risk.
* **Wrong source set:** vendor or generated code dominates the report. Configure includes and exclusions deliberately.
* **Driver drift:** Xdebug or PCOV versions change results or performance. Pin and document the driver.
* **Uncovered critical branch:** a percentage hides an authorization or rollback path. Review risk-focused reports.
* **Slow CI:** collecting full path detail blocks feedback. Separate fast and comprehensive jobs.

## Exercises

1. Generate line and branch coverage for one bounded context and list the five most important uncovered branches.
2. Add a CI floor for changed source, then document a legitimate exclusion and its review date.
3. Take a test with high line coverage and improve its assertions for invalid input, authorization denial, and side-effect absence.
4. Compare coverage collection with Xdebug and PCOV on the same suite and record the performance trade-off.

## Review Questions

1. What does line coverage fail to tell you?
2. Why can path coverage become impractical?
3. Which branches deserve attention before a global percentage?
4. How can a threshold encourage bad tests?
5. When is an exclusion legitimate?
6. What evidence complements coverage for assertion quality?

## Summary

Coverage reports where tests execute code; they do not prove that behavior is correct. Understand line, branch, condition, function, and path measures, configure the intended source, use thresholds as floors, investigate risk-heavy gaps, document exclusions, and pair coverage with strong assertions and mutation testing.

## References

- [PHPUnit Code Coverage](https://docs.phpunit.de/en/11.5/code-coverage.html)
- [PHPUnit Configuration](https://docs.phpunit.de/en/11.5/configuration.html)
- [Xdebug Code Coverage](https://xdebug.org/docs/code_coverage)
- [PCOV](https://github.com/krakjoe/pcov)

