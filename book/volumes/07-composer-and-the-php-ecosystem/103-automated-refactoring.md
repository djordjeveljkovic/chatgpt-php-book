---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 103
title: Automated Refactoring
slug: automated-refactoring
status: complete
summary: ../../_ai/chapter-summaries/103-automated-refactoring-summary.md
---

# Chapter 103 — Automated Refactoring

## Why This Matters

Changing a PHP codebase safely is often repetitive. A language upgrade can require the same source adjustment in hundreds of files. A renamed API may have callers across several packages. A legacy method may need to become a typed property, or a framework migration may replace one registration pattern with another. Doing every edit by hand is slow and inconsistent; doing a broad search-and-replace can damage code that only looks similar.

Automated refactoring tools apply structured transformations across files. They can make a large change more consistent and help a team modernize code incrementally. But a transformation is still a proposed change to an executable program. Tool output must be scoped, reviewed, tested, and tied to a known tool and rule set. A dry run is a preview, not evidence that the resulting behavior is correct.

[Chapter 102](102-formatting.md) covers formatting rules that mostly normalize presentation. This chapter covers source transformations, from narrow syntax rewrites to framework or PHP-version migrations. [Chapter 101](101-static-analysis.md) explains static analysis, which can help identify affected code but does not itself refactor it.

## Mental Model

An automated refactoring is a mapping from one program representation to another:

```text
source + configuration + tool version
                 ↓
         proposed source changes
                 ↓
     reviewed and tested program
```

The change is only as reliable as its preconditions, transformation logic, and coverage of the code that depends on the affected behavior. A tool can parse a call into a syntax tree and rename it precisely, yet still miss reflective access, dynamically constructed method names, serialized class names, configuration strings, or code generated at runtime.

Tools have different roles. A formatter changes layout and selected style constructs. A static analyzer reports likely defects under a model of the program. A migration or refactoring tool rewrites code according to rules. A test suite checks selected runtime behavior. A single tool may offer several capabilities, but their assurances remain different.

## Kinds of Transformations

Automated edits vary in how much judgment they require:

| Transformation | Example | Main concern |
| --- | --- | --- |
| Presentation | Consistent indentation or import spacing | Does the output match the style policy? |
| Mechanical syntax | `array(...)` to `[...]` | Does this syntax work on every supported PHP version? |
| API migration | Replace a deprecated method with its successor | Are arguments, return values, and side effects equivalent? |
| Type or design change | Infer a property type or extract a service | Does the inferred contract match all runtime states? |
| Framework migration | Replace registration or lifecycle patterns | Does the framework version and application boot behavior remain compatible? |

The further a transformation moves from presentation toward program design, the more it depends on repository-specific facts. A tool can confidently normalize braces based on syntax. It cannot infer the correct business meaning of a nullable field merely from its current examples.

## Token and Syntax-Aware Tools

Text replacement sees characters. A parser or tokenizer can distinguish a method call from a comment or string and can preserve the surrounding syntax. Tools such as [Rector](https://getrector.com/documentation) analyze PHP source and apply rules to syntax structures; PHP-CS-Fixer has rules that also perform certain code modernization tasks.

Syntax awareness reduces accidental matches, but it does not guarantee semantic equivalence. A transformation may be correct only when assumptions hold, such as a property always being initialized in every constructor path. Dynamic dispatch and runtime conventions can hide uses that are not visible in the source tree. A rule can also be correct in one framework release and wrong in another. Treat its assumptions as part of the migration plan.

## Example: A Narrow Rector Rule

Rector supports applying selected rules or groups of rules. The project's development dependency should be locked, and configuration should limit the paths and transformations under consideration. For example, a project might evaluate the documented `TypedPropertyFromStrictConstructorRector` rule on application and test code:

Rector describes this rule as adding a typed property when the constructor assigns a value from a strictly typed parameter. A simplified transformation looks like this:

```php
<?php

class SomeObject
{
    private $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }
}
```

```php
<?php

class SomeObject
{
    private string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }
}
```

The constructor is evidence for the initial assignment, but it does not show every later write. Search for assignments from hydration, reflection, unserialization, inheritance, or other methods; a new declaration may turn a value the old program tolerated into a runtime `TypeError`. Also check that the property's initialization lifecycle is compatible with a typed property and the project's minimum PHP version.

```php
<?php

declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\TypeDeclaration\Rector\Property\TypedPropertyFromStrictConstructorRector;

return RectorConfig::configure()
    ->withPaths([
        __DIR__ . '/src',
        __DIR__ . '/tests',
    ])
    ->withRules([
        TypedPropertyFromStrictConstructorRector::class,
    ]);
```

First ask what the rule assumes about initialization, inheritance, and the project's supported PHP versions. Then preview the changes:

```sh
vendor/bin/rector process --dry-run
```

After inspecting the proposed diff, apply the same configured rule without `--dry-run` only on a working branch. The rule name indicates its intended transformation, not that every inferred type is a domain guarantee. If the constructor can receive invalid or partial data, a native type declaration will not replace the validation the application needs.

Rulesets can be useful when migrating by PHP version or framework release, but a set may contain many transformations. Start with a narrow scope or one migration step, inspect changed files, and expand only after the previous output is understood. Rector's [integration guidance](https://getrector.com/documentation/integration-to-new-project) recommends gradual, reviewable steps for legacy upgrades.

## A Safe Migration Workflow

Use a workflow that keeps cause and effect visible:

1. **Define the intended outcome.** Record the affected API, PHP/framework versions, supported runtime range, and behavior that must stay stable.
2. **Establish a baseline.** Start from a clean worktree. Run the relevant tests, syntax checks, static analysis, and application-specific smoke tests. Record existing failures rather than attributing them to the migration.
3. **Pin the tool and configuration.** Install it as a project development dependency, update the lock file, commit the rule configuration, and limit paths deliberately.
4. **Preview a bounded change.** Run dry-run mode, inspect representative changes, and check whether the rule's preconditions hold in this codebase.
5. **Apply in small slices.** Separate rules, directories, or release steps into reviewable changes. Avoid combining a dependency upgrade, style cleanup, and behavior change in one giant diff.
6. **Review the diff.** Look for missed dynamic uses, generated or excluded source, changed defaults, new declarations, altered exception behavior, and surprising formatting.
7. **Verify the result.** Run syntax checks for the supported PHP target, tests, static analysis, and migration-specific checks. Add tests for behavior that the transformation assumes.
8. **Commit and observe.** Keep the migration diff traceable. For a change that can affect runtime behavior, monitor deployment and preserve a practical rollback path.

Do not use a tool's exit code as the only review. A successful command means it completed according to its own rules, not that the repository's behavior is preserved.

## Legacy Code and Characterization

An old codebase may lack reliable tests or clear contracts. Before changing its implementation, add characterization tests around the behavior callers rely on. These tests can preserve odd behavior that the team has not yet decided to change. [Chapter 272](../../volumes/18-legacy-php/272-characterization-tests.md) covers this technique in depth.

When a rule cannot confidently transform a dynamic pattern, leave that code for deliberate manual work and document the exception. For example, a package may call a method through a string assembled from configuration, or a framework may register handlers by naming convention. A syntax-based tool cannot safely infer every such connection. Search the repository and runtime configuration, add targeted tests, and update those call sites explicitly.

Automated refactoring is also useful when introducing types to a gradual codebase. An inferred type is a candidate contract. Check nullable paths, hydration and serialization, reflection, inheritance, database values, and external input. Add runtime validation where the boundary requires it. [Chapter 101](101-static-analysis.md) explains why type annotations and analyzer models do not validate data at runtime.

## Version and Compatibility Boundaries

A migration must respect the minimum PHP version in `composer.json`, the PHP runtime used in CI and production, and the dependency versions installed from the lock file. A tool may emit syntax accepted by the interpreter running the tool but unsupported by one of the application's deployed runtimes. Set the tool's target version where available and test the lowest supported version.

Framework migrations have another boundary: application behavior can depend on container registration, middleware order, event timing, boot hooks, or defaults that changed between releases. A syntactically valid rewrite may alter this lifecycle. Consult the framework's official migration guide and compare behavior at the integration boundary rather than assuming similarly named APIs are interchangeable.

Tool versions matter too. A new release may add rules, change type inference, or alter the default set. Upgrade the refactoring tool independently where practical. Review configuration and output changes, then rerun the full relevant verification suite. The tool's lock-backed version is part of the reproducible migration record.

## Large Transformations and Git

For a repository-wide migration, coordinate the work with active branches. A large rewrite can create merge conflicts even if it preserves behavior. Consider a short-lived migration branch, a generated patch split by directory, or staged pull requests that first establish compatibility helpers and then migrate callers.

Keep the original and transformed versions easy to compare. Avoid reformatting unrelated files, regenerating lock files without a dependency reason, or modifying generated artifacts as if they were source. If generated code must change, update its source and use the generator's normal process. The review should make it clear which files are authoritative.

When the transformation is itself the intended behavior change, update or add tests that describe the new contract. If the refactor should preserve behavior, tests should compare the old and new externally observable behavior where practical. Tests are not a proof for every possible input, but they provide evidence for the invariants most likely to be broken.

## CI and Ongoing Use

CI can run a refactoring tool in dry-run mode to detect code that no longer conforms to a migration target or required modernization rule. This can be helpful when the codebase is gradually adopting a rule. Keep the command scoped and deterministic so developers can reproduce the proposed change locally.

Do not normally run an applying migration automatically during a build. CI should report a difference and fail clearly; it should not mutate a checkout that the developer cannot review or commit. For one-off migrations, perform the rewrite on a branch and keep the tool configuration or exact command in the change record. If the tool remains in CI, maintain its configuration and explain what the enforced rule means.

## Testing

The relevant verification depends on the transformation. At minimum, syntax-check the changed files with the project's supported PHP version and run tests that cover affected behavior. For API or framework migrations, add integration tests that exercise the real container, persistence, HTTP, or event boundary involved. Run static analysis after a type-oriented transformation, while remembering that analysis validates a model rather than runtime input. If possible, compare serialized output, emitted events, exceptions, or database writes before and after a behavior-preserving rewrite.

For a custom Rector rule or codemod, build fixtures with expected before-and-after source, then execute the transformed example and test the relevant behavior. Include negative cases where the rule must leave code unchanged. This catches overbroad matching and documents the transformation's preconditions.

## Common Mistakes

- Applying a broad ruleset before inspecting what it will change.
- Treating a dry-run diff as proof of semantic equivalence.
- Using text replacement for syntax that can appear in comments, strings, or different contexts.
- Trusting inferred types without checking nullability, inheritance, hydration, and external input.
- Ignoring dynamic calls, reflection, configuration, generated source, and framework conventions.
- Combining a migration with unrelated formatting, dependency, and feature changes.
- Running a tool version locally that differs from the one used in CI.
- Testing only the highest PHP version while the project supports older versions.
- Assuming all changes are reversible because they were generated automatically.
- Keeping temporary migration configuration in CI without documenting its purpose and scope.

## Senior Engineer Thinking

Automation is valuable when it reduces repetitive work while preserving reviewability. The tool should be a constrained assistant: its version, configuration, target files, assumptions, and output are visible. The team supplies the missing context through tests, code review, migration guides, and operational observation.

Choose the narrowest transformation that moves the code toward the goal. A predictable, staged migration is easier to diagnose and roll back than a large collection of simultaneous changes. When the tool cannot see a runtime convention, make that boundary explicit and handle it with evidence rather than forcing a rewrite.

## Exercises

1. Configure Rector with one narrowly selected rule and a limited path. Review its dry-run output, then run the migration on a branch and verify it with tests.
2. Pick a deprecated API in a small project. Find all call paths, write a test for the replacement behavior, migrate the code, and explain any dynamic usages the tool cannot detect.
3. Build a tiny codemod or custom Rector rule with positive and negative fixtures. Show that it changes only the intended syntax and preserves runtime behavior.
4. Plan an upgrade for a legacy codebase with incomplete tests. Identify characterization tests, migration slices, supported PHP versions, and a rollback point.
5. Compare a format-only rule with a behavior-oriented migration rule. Explain why their review and verification requirements differ.

## Review Questions

1. What information can an AST-based tool use that a plain text replacement cannot?
2. Why does syntax-aware transformation still fail to guarantee semantic equivalence?
3. What belongs in a migration baseline before running an automated refactor?
4. Why should broad transformations be applied in small, reviewable steps?
5. Which runtime or repository conventions can hide usages from a refactoring tool?
6. How should the lowest supported PHP version influence a generated migration?
7. What tests are useful for a custom codemod?
8. When should migration tooling run in CI, and why should CI generally report rather than apply changes?

## Summary

Automated refactoring tools can make large source changes consistently, but their output depends on tool version, configuration, visible syntax, and assumptions about the application. Pin the tool, define a bounded scope, preview changes, review the diff, and verify behavior against the project's full supported runtime range. Use characterization tests where legacy behavior is unclear and integration tests where framework or external boundaries may change.

A dry run is a review aid, not a correctness proof. Keep transformations small enough to understand, and make exceptional dynamic behavior visible for manual treatment. This completes Volume VII; the next volume begins with database fundamentals in [Chapter 104](../../volumes/08-databases/104-sql-for-php-developers.md).

## References

- [Rector documentation](https://getrector.com/documentation)
- [Rector: integrating Rector into a new project](https://getrector.com/documentation/integration-to-new-project)
- [Rector: custom rules](https://getrector.com/documentation/custom-rule)
- [PHP-CS-Fixer documentation](https://github.com/PHP-CS-Fixer/PHP-CS-Fixer)
- [PHP Manual: Type declarations](https://www.php.net/manual/en/language.types.declarations.php)
