---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 101
title: Static Analysis
slug: static-analysis
status: complete
summary: ../../_ai/chapter-summaries/101-static-analysis-summary.md
---

# Chapter 101 — Static Analysis

## Why This Matters

A test suite can verify behavior only for the inputs and execution paths that it exercises. Production code still contains paths that tests miss: a nullable value reaches a method that expects an object, an associative array lacks a key, or a refactor changes a return type used elsewhere. Static analysis can inspect code and report many such inconsistencies without running the application's normal execution paths.

For PHP teams, tools such as PHPStan and Psalm turn declared types, PHPDoc, source code, and configuration into a model of how values flow through a codebase. Their value is not that they make a project “type safe” by command. Their value is a fast feedback loop: find an inconsistency near the code change, make the invariant explicit, and keep later changes from silently weakening it.

Static analysis has limits. PHP is dynamically typed, programs can use reflection and magic dispatch, external data has unknown shape until validated, and annotations can be wrong. The analyzer can reason only from the code and model it sees. Treat its diagnostics as evidence to investigate, not as proof that the application is correct or secure.

This chapter focuses on analysis of types and program behavior. [Chapter 102](102-formatting.md) covers formatting and coding-style tools; [Chapter 103](103-automated-refactoring.md) covers automated code transformations. They can all run in CI, but they answer different questions.

## Mental Model

Static analysis builds an approximate model of code before the application's ordinary runtime executes it:

```text
PHP source + native declarations + PHPDoc + configuration/stubs
                              ↓
                  analyzer builds a code model
                              ↓
               diagnostics about possible violations
                              ↓
         engineer checks the claim, fixes or documents it
```

The source and annotations are inputs to the model. The model is not the running application, and a clean report means only that the chosen analyzer found no reportable violations in the selected files under its configured rules and assumptions.

Static analysis differs from three other checks:

- `php -l` asks whether PHP can parse a file. It does not check that calls or data flows make sense.
- Tests execute selected paths with selected data and assert observed behavior.
- Formatting and linting tools enforce layout or selected source conventions; they do not usually infer the same value-flow properties as a type analyzer.

Use all the checks that address real risks. Static analysis complements tests by examining many possible paths cheaply, while tests verify runtime behavior, external integrations, and important domain invariants.

## PHP Types and the Runtime Boundary

PHP is dynamically typed: a variable's type follows its current value. Native type declarations let a program state contracts for parameters, return values, properties, and other supported declarations. PHP checks those declarations at runtime and can throw a `TypeError` when a value does not satisfy the applicable declaration. Scalar coercion can also depend on `strict_types` at the call site, so a declaration is not always the same as a static guarantee. See the PHP Manual's [type declarations](https://www.php.net/types.declarations) and [type-system introduction](https://www.php.net/manual/en/language.types.intro.php).

Native declarations still leave many useful invariants unstated. A parameter typed `array` can contain any keys and values. A `string` may be empty, malformed, or outside the domain expected by the business rule. A nullable type can say a value is either an object or `null`, but not that the object belongs to a particular tenant. Runtime declarations catch violations at specific call, return, or write boundaries; they do not automatically validate every external value or prove the program's logic.

An analyzer can reason about declarations across call sites and control flow. It may report that a nullable result is dereferenced without a null check, or that a caller passes an incompatible value. It can also incorporate PHPDoc types and tool-specific rules. The analysis is still performed before execution and depends on what the analyzer can discover about functions, classes, extensions, framework conventions, and dynamic behavior.

## From a Broad Array to a Useful Type

Suppose an endpoint receives rows decoded from JSON and passes them to a notification service. A native `array` declaration says almost nothing about the internal shape:

```php
<?php

declare(strict_types=1);

/**
 * @param list<array{id: int, email: string}> $records
 * @return list<string>
 */
function extractEmails(array $records): array
{
    $emails = [];

    foreach ($records as $record) {
        $emails[] = $record['email'];
    }

    return $emails;
}

var_export(extractEmails([
    ['id' => 1, 'email' => 'reader@example.test'],
]));
```

The PHP runtime checks that `$records` and the return value are arrays. It does not enforce `list<array{id: int, email: string}>`. PHPStan and Psalm understand common PHPDoc extensions for generic collections, lists, and array shapes; their [PHPDoc type references](https://phpstan.org/writing-php-code/phpdoc-types) and [array-type documentation](https://psalm.dev/docs/annotating_code/type_syntax/array_types/) describe supported syntax. The `list` states that the outer array has consecutive integer keys; the shape states which fields each record is expected to contain.

That precise annotation helps the analyzer catch a misspelled key or a caller passing an incompatible array. It does not transform untrusted JSON. Before calling this function with decoded input, validate that every item is an array, that required fields exist, and that their values have the expected types. The annotation should describe a fact established by code or by a trusted, tested boundary—not a wish about incoming data.

## Generics and Stubs

PHPDoc can express relationships that a plain native type cannot. A generic function can preserve the element type of a list through an operation; a generic collection can say that it stores one particular value type. A shape type can describe a record-like array with named required and optional keys. Together, these annotations reduce the spread of `mixed` and make callers more informative to the analyzer.

For example, a reusable `firstOrNull()` function can preserve the input element type in its result:

```php
<?php

declare(strict_types=1);

/**
 * @template T
 * @param list<T> $items
 * @return T|null
 */
function firstOrNull(array $items): mixed
{
    return $items[0] ?? null;
}

var_dump(firstOrNull([1, 2]));
```

The PHP signature only promises an array input and a `mixed` result at runtime. The analyzer interprets `@template T` and relates the result to the input list. It can therefore know that `firstOrNull([new User()])` returns a `User|null` in the analyzed program. Native PHP does not enforce the generic argument or the relation described by PHPDoc. Type annotation dialects also differ between tools; use the chosen analyzer's documentation and avoid assuming every IDE, analyzer, or documentation generator interprets every extension identically.

Stub files extend an analyzer's model when third-party code has missing or inaccurate type information, or when a dependency cannot be edited. A stub describes symbols the analyzer should associate with external code; it does not change the vendor package's runtime implementation. PHPStan documents stubs as a way to override third-party PHPDoc, while Psalm supports stubs for describing code that is unavailable to its scanner or needs additional type information. See [PHPStan stub files](https://phpstan.org/user-guide/stub-files) and [Psalm configuration](https://psalm.dev/docs/running_psalm/configuration/).

Keep a stub narrow, version-controlled, and tied to the dependency behavior it describes. A stub is an assertion supplied to the analyzer. If the library later changes its real behavior while the stub remains unchanged, the report can look precise while being wrong. Review stubs when upgrading the dependency and add integration tests around the boundary.

## What an Analyzer Can and Cannot Establish

Static analysis is most useful for properties that can be inferred from a code model: a method call is valid for the inferred receiver type, a required array key may be absent, a value may be null, a return value may violate a declared contract, or a branch may be impossible under the inferred conditions. Some tools also provide configurable rules and security-oriented data-flow checks. Which diagnostics exist depends on the analyzer, its configuration, extensions, and version.

Do not describe an ordinary PHPStan or Psalm run as a general mathematical proof. PHP code can load classes dynamically, call magic methods, use global state, inspect runtime configuration, or receive data from a database, message broker, or HTTP request. An analyzer may model some of those features through framework extensions and stubs, but it cannot infer facts that have not been declared, modeled, or checked. Even in mostly static code, a claimed type can be inaccurate and behavior can be wrong while all types line up.

It helps to distinguish two failure directions:

- A **false positive** is a report whose warning does not represent a real defect in the application's actual behavior. Dynamic framework registration that the analyzer does not understand is one common cause. Add the missing model when possible; otherwise suppress only that specific diagnostic with a reason.
- A **false negative** is a defect that the analyzer does not report. It may occur because the relevant code is excluded, the analyzer does not model a dynamic path, a baseline suppresses the issue, or an annotation asserts an incorrect fact.

There is a trade-off between stricter analysis and the amount of code or ecosystem behavior the tool must understand. Raising strictness may expose weak types and bugs but can also reveal gaps in framework support. Relaxing rules reduces noise but can leave defects invisible. The goal is not a perfect score; it is a trustworthy signal that developers will investigate.

## Gradual Adoption in an Existing Codebase

Introducing strict analysis to a large legacy application all at once can produce a backlog that no team can reasonably fix in one change. A baseline provides a way to start checking new work while recording existing diagnostics. PHPStan and Psalm both document baseline workflows, but their formats and commands differ; see the official [PHPStan baseline guide](https://phpstan.org/user-guide/baseline) and [Psalm issue-handling guide](https://psalm.dev/docs/running_psalm/dealing_with_code_issues/).

A practical rollout is:

1. Choose the application code and boundaries to analyze. Exclude generated files and vendor source unless there is a specific reason to inspect them.
2. Configure the analyzer for the PHP version and extensions the project supports, and teach it about framework bootstrapping or magic behavior it must understand.
3. Run the analysis and inspect representative reports. Fix configuration gaps and high-value defects before baselining the remaining legacy findings.
4. Commit the baseline and analyzer configuration. Make CI fail on newly reported violations under the committed configuration.
5. Add types and runtime validation where they communicate or enforce real invariants. Reduce the baseline as old findings are fixed.
6. Increase strictness or analysis scope in controlled steps, and review the effect when upgrading the analyzer.

A baseline is a debt ledger, not a type improvement. It silences listed findings, so developers should not regenerate it automatically whenever CI fails. Keep it reviewable, prevent unexplained growth, and periodically report its size and age. For local suppressions, use the narrowest supported scope and leave a short explanation of the invariant or tool limitation that justifies it.

## Configure a Reliable CI Feedback Loop

Install the analyzer as a project development tool and control the resolved tool version in CI. Applications can do this with their committed Composer lock file; a library can use a lock file for its own development and tests, as [Chapter 95](095-composer-lock.md) explains. [Chapter 94](094-composer-json.md) covers development dependencies and scripts. Add a script that runs the chosen analyzer using a checked-in configuration. For example, with a project-local installation:

```sh
vendor/bin/phpstan analyse src tests
```

or:

```sh
vendor/bin/psalm
```

Check each tool's current documentation for its configuration and options; do not assume their commands, type dialects, baseline behavior, or defaults are interchangeable. PHPStan's [command-line guide](https://phpstan.org/user-guide/command-line-usage) documents `analyse`; Psalm's [configuration guide](https://psalm.dev/docs/running_psalm/configuration/) describes the files and project paths it scans.

Run analysis against the supported PHP target, the same source directories, configuration, bootstrap, and generated classes in developer workflows and CI. Analyzer upgrades can add or change diagnostics, so upgrade in an intentional dependency change and review any baseline or configuration differences. Use caches to reduce repeat work where supported, but do not allow stale caches or a developer-only ignore rule to make CI disagree with the submitted source. A green CI result is useful only if CI analyzed all intended code and failed on the diagnostics the team considers actionable.

For large projects, analysis time and memory are real operational costs. Start with a representative scope, measure runtime, and use the analyzer's supported result cache or CI cache. Avoid both extremes: scanning vendor and generated artifacts indiscriminately can slow the feedback loop, while analyzing only touched files can miss cross-file effects. If you split work into modules or jobs, keep a full-project analysis in the normal merge/release path at a frequency appropriate to the project.

## Bad and Better Responses to a Finding

Suppose an analyzer reports that an array key may be missing. A weak response is to add `@var string $email` at the use site to silence the error. That annotation does not prove the key exists; it merely tells the analyzer to trust the assertion.

A better response depends on the actual boundary:

- If input may legitimately omit the value, make the type nullable or optional and handle absence.
- If the value is required, validate it at the input boundary and pass a validated object or well-described structure inward.
- If the framework guarantees the value but the analyzer cannot discover the guarantee, add a narrow framework extension or stub and test that integration.
- If the report is a genuine tool limitation, suppress only that issue and explain why the runtime behavior is safe.

Types are most valuable when they describe facts. Do not use a PHPDoc assertion to invent a fact that the code has not established.

## Testing

Static analysis should run beside tests, not instead of them. Tests are needed for domain rules, persistence behavior, serialization, race conditions, external services, and runtime configuration. Analysis can catch mismatched assumptions between tested components and can inspect many call paths, but it does not execute SQL against your production schema, prove that a remote service honors a contract, or guarantee that a business invariant holds.

Use analysis findings to add targeted tests when they reveal an edge case. For example, a possible null value suggests a test for the absent value; a shape mismatch suggests tests for missing and malformed request fields. Then add the runtime validation or design change that makes the contract true. This closes the gap between the analyzer's static model and the values the program actually receives.

## Common Mistakes

- Treating a clean analyzer run as proof of correctness, security, or production readiness.
- Assuming native `array`, `string`, or class declarations enforce every domain invariant.
- Treating PHPDoc generics and array shapes as runtime validation.
- Writing a shape annotation for untrusted decoded data before validating that data.
- Adding broad `mixed` or `@var` assertions simply to eliminate diagnostics.
- Generating a baseline and never reviewing or shrinking it.
- Silencing a finding globally when one local integration gap is responsible.
- Upgrading an analyzer without checking newly surfaced errors or changed assumptions.
- Running analysis only on changed files and missing callers elsewhere in the project.
- Confusing static analysis with formatting, syntax checking, tests, or automatic refactoring.

## Senior Engineer Thinking

Static analysis is a communication system between a codebase and its maintainers. Native declarations state enforceable runtime boundaries. PHPDoc can carry richer intent to compatible tools. Runtime validation turns untrusted values into trusted domain data. Tests verify selected behavior. Analyzer configuration, stubs, and baselines connect these layers, but each can also create false confidence when it drifts from reality.

Build a signal developers trust: analyze meaningful code, model dynamic framework behavior deliberately, keep suppressions narrow, and ratchet legacy debt downward. Measure whether findings are caught early and whether they lead to better boundaries. A useful analysis pipeline catches a class of defects while leaving the team responsible for understanding the behavior it cannot prove.

## Exercises

1. Add an analysis tool to a small PHP project and configure it to inspect the application source and tests. Record one useful report, one configuration issue, and the command that CI will run.
2. Take a function that accepts a generic `array`. Define its key and value shape with PHPDoc, then write runtime validation for data originating outside the process. Explain which guarantees come from the runtime and which are analyzer assumptions.
3. Write a generic `firstOrNull()` helper and inspect how the analyzer infers its result for lists of integers and lists of domain objects. Remove the generic annotation and compare the information lost.
4. Find a third-party function with missing or inaccurate type information. Add a narrow analyzer stub, then write a test that would detect if the real dependency no longer matches the stub.
5. Generate a baseline for a legacy directory. Make one existing issue disappear and introduce a new one. Confirm that CI catches the new issue, then remove the fixed baseline entry.
6. Create a dynamic framework call that the analyzer cannot infer. Compare fixing it with runtime validation, configuring an extension/stub, and a local suppression. State the cost and maintenance risk of each choice.
7. Write a short team policy that distinguishes analyzer findings, formatter changes, test failures, and automated refactor output in code review.

## Review Questions

1. What does static analysis inspect, and how does it differ from `php -l` and tests?
2. What do native PHP type declarations enforce at runtime, and what do they leave unspecified?
3. Why are PHPDoc generics and array shapes useful, and why are they not runtime guarantees?
4. When should a project use a stub file, and what risk does an inaccurate stub create?
5. What is a baseline, and why should it be treated as technical debt?
6. How can an analyzer produce false positives and false negatives?
7. Why should an analyzer's target PHP version and framework model match the project environment?
8. What is the danger of suppressing a diagnostic globally or adding an unsupported `@var` assertion?
9. Why do analyzer upgrades deserve deliberate review?
10. Which correctness questions still require tests or runtime checks?

## Summary

Static analysis checks code against a model built from PHP source, native types, PHPDoc, stubs, and configuration. It can report many likely type and control-flow defects before affected paths execute, but it cannot establish overall correctness. PHP's runtime declarations enforce only their declared contracts at runtime; richer PHPDoc types such as generics and array shapes guide compatible analyzers without validating runtime values.

Adopt analyzers gradually with a checked-in configuration, an explicit baseline for existing debt, and CI that examines the intended code. Keep stubs accurate, suppressions narrow, and baseline growth visible. Validate external data at runtime, test domain and integration behavior, and treat analyzer results as a valuable feedback signal with known limits. Formatting and automated refactoring are separate concerns covered in Chapters 102 and 103.

## References

- [PHP Manual: Type declarations](https://www.php.net/types.declarations)
- [PHP Manual: Introduction to types](https://www.php.net/manual/en/language.types.intro.php)
- [PHPStan: PHPDoc types](https://phpstan.org/writing-php-code/phpdoc-types)
- [PHPStan: Generics in PHP using PHPDocs](https://phpstan.org/blog/generics-in-php-using-phpdocs)
- [PHPStan: Stub files](https://phpstan.org/user-guide/stub-files)
- [PHPStan: Baseline](https://phpstan.org/user-guide/baseline)
- [PHPStan: Command-line usage](https://phpstan.org/user-guide/command-line-usage)
- [Psalm: Array types](https://psalm.dev/docs/annotating_code/type_syntax/array_types/)
- [Psalm: Configuration and stubs](https://psalm.dev/docs/running_psalm/configuration/)
- [Psalm: Dealing with code issues and baselines](https://psalm.dev/docs/running_psalm/dealing_with_code_issues/)
- [Chapter 94 — `composer.json`](094-composer-json.md)
- [Chapter 95 — `composer.lock`](095-composer-lock.md)
- [Chapter 102 — Formatting](102-formatting.md)
- [Chapter 103 — Automated Refactoring](103-automated-refactoring.md)
