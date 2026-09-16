---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 102
title: Formatting
slug: formatting
status: complete
summary: ../../_ai/chapter-summaries/102-formatting-summary.md
---

# Chapter 102 — Formatting

## Why This Matters

Two correct PHP implementations can still impose different reading costs. One uses tabs, another spaces; one wraps long calls, another leaves them on one line; imports and braces appear in different orders. In isolation these choices rarely break a request. Across a team and a long-lived codebase, however, inconsistent presentation makes review slower and makes meaningful changes harder to distinguish from personal preference.

A shared style policy moves routine layout decisions out of code review. A formatter can apply that policy consistently, while a check in continuous integration (CI) prevents accidental drift. The team can then spend review time on behavior, data boundaries, and failure handling. Formatting does not make code correct, and a style guide does not settle every design question. It establishes a predictable visual surface.

[Chapter 100](100-psr-standards.md) describes PHP-FIG's style recommendations. This chapter focuses on how a project selects and enforces a style. [Chapter 101](101-static-analysis.md) covers analyzer findings about types and program behavior, while [Chapter 103](103-automated-refactoring.md) covers transformations that change source structure or meaning.

## Mental Model

Treat code style as a versioned project contract:

```text
team chooses a style target
          ↓
repository records tool and configuration
          ↓
developers format before review
          ↓
CI checks that the submitted code matches
```

The tool is an implementation of the policy. It discovers files, applies configured rules, and either rewrites them or reports differences. PHP itself does not know whether a team prefers one brace layout or another. The formatter does not know whether the code expresses the right business rule.

Keep the purposes distinct:

| Check | Main question |
| --- | --- |
| `php -l` | Can PHP parse this file? |
| Formatter or style checker | Does the source follow the configured presentation and style rules? |
| Static analyzer | Do the modeled types and control flows contain likely defects? |
| Tests | Does selected behavior match expected behavior in the exercised environment? |
| Refactoring tool | Can a configured transformation change the source in a particular way? |

One command may perform more than one kind of check, but these questions are not interchangeable. A clean formatting check says nothing about whether an authorization condition is correct.

## Choose a Style Target

For shared PHP code, PSR-12 remains an accepted PHP-FIG extended coding style guide. The newer [Coding Style PER](https://www.php-fig.org/per/coding-style/) extends and replaces PSR-12, requires PSR-1, and clarifies style for newer PHP syntax. As [Chapter 100](100-psr-standards.md) explains, a PER is intended to evolve; check its current release and status when adopting it.

Choose a target the team can implement and maintain. A library may prefer a widely recognized baseline that reduces friction for contributors. An application may already have conventions that make migration more expensive than the value of changing them. A framework or organization may publish a compatible style. In every case, write down the chosen target and the exceptions that matter. “Use the team's usual style” is not a reliable instruction to a new contributor or a tool.

The formatter's ruleset must correspond to the policy. A formatter may offer a named ruleset for PSR-12, PER Coding Style, or a community convention, but the set may trail the latest text or omit rules the tool cannot implement. The PHP-CS-Fixer documentation, for example, describes its [`@PER-CS` rule set](https://github.com/PHP-CS-Fixer/PHP-CS-Fixer/blob/master/doc/ruleSets/PER-CS.rst) as an alias for its latest supported PER-CS rules. Review the exact tool version and rule set instead of assuming that a label proves full conformance.

## Formatter and Checker Roles

A formatter rewrites source to match its configured rules. A checker reports violations without changing tracked files. Some tools provide both modes; others separate detection and fixing.

[PHP-CS-Fixer](https://github.com/PHP-CS-Fixer/PHP-CS-Fixer) supports configurable rulesets and can check or rewrite files. [PHP_CodeSniffer](https://github.com/PHPCSStandards/PHP_CodeSniffer) tokenizes PHP and reports violations against a ruleset; its companion `phpcbf` attempts to fix supported violations. The exact set of fixable findings depends on the selected sniffs. A team may use one tool or combine tools when they detect different classes of policy violations. Two tools that enforce overlapping rules can also disagree, so designate one output as authoritative for every shared rule.

Avoid making the tool choice itself the policy. Store the rules and included paths in the repository, explain any local conventions, and pin the development dependency in Composer. [Chapter 94](094-composer-json.md) covers development dependencies and scripts; [Chapter 95](095-composer-lock.md) explains how the lock file makes application tooling reproducible.

## A Project Configuration

Install the selected formatter as a development dependency and commit the resulting Composer manifest and lock changes:

```sh
composer require --dev friendsofphp/php-cs-fixer
```

A PHP-CS-Fixer distribution configuration can select the project's PHP files and a style ruleset. This example uses the tool's current PER-CS alias; a team that wants a deliberately frozen ruleset should select the supported versioned ruleset documented by its pinned formatter release.

```php
<?php

declare(strict_types=1);

$finder = (new PhpCsFixer\Finder())
    ->in(__DIR__ . '/src')
    ->in(__DIR__ . '/tests');

return (new PhpCsFixer\Config())
    ->setRiskyAllowed(false)
    ->setRules([
        '@PER-CS' => true,
    ])
    ->setFinder($finder);
```

The finder deliberately excludes `vendor/`, generated files, caches, and build output. Apply the tool to files the project owns. If templates contain embedded PHP, check whether the formatter correctly supports their mixed-language syntax before including them.

Repository text settings help editors agree on whitespace and line endings. They complement the PHP formatter but do not replace its PHP-specific rules:

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.php]
indent_style = space
indent_size = 4
```

The policy can be exposed through Composer scripts so contributors use the same command locally and in CI:

```json
{
  "scripts": {
    "format": "php-cs-fixer fix",
    "format:check": "php-cs-fixer fix --dry-run --diff"
  }
}
```

The mutating command is for local use; the check command reports the diff and leaves the worktree untouched. Exact command options vary between formatter releases. Verify them against the installed version and keep CI's command in sync with the checked-in configuration.

## Introducing a Formatter to Existing Code

Running a formatter over a large legacy repository may produce thousands of changed lines while changing no intended behavior. That diff is hard to review and creates conflicts with branches already in flight. Separate the mechanical baseline from functional work:

1. Select the policy and pin the formatter version.
2. Run the formatter against the intended paths and review a representative sample. Check templates, generated files, and unusual syntax separately.
3. Apply the initial formatting in a dedicated change. Do not mix it with feature work or a dependency upgrade.
4. Run syntax checks and the project's tests. Review the diff for accidental file changes or rules that alter more than layout.
5. Merge the baseline and then require formatting checks for new changes.

If a full-repository cleanup is not practical, format touched files and establish a clear rule for new code. A permanent path exclusion or growing list of per-file exceptions can become invisible debt, so record why it exists and revisit it. Avoid having CI rewrite files during a build: a contributor should see and commit the resulting changes locally.

For a normal change, run the formatter before staging and inspect the resulting diff:

```sh
composer format
git diff --check
git diff -- src/InvoiceService.php
composer format:check
```

`git diff --check` catches whitespace errors such as trailing whitespace; it is not a replacement for the configured style check. Keep a style-only diff recognizable. If a formatter changes files unrelated to the task, investigate the path selection or tool version rather than hiding the noise in a functional commit.

## Editor and CI Feedback

Editor integrations can format on save or expose style diagnostics while typing. They should use the repository's formatter version and configuration when possible. A developer's global configuration is convenient but must not be the only source of truth: it may differ from CI or another contributor's editor.

CI should run the same non-mutating check against the same source paths. A useful failure includes the files and proposed changes so a developer can reproduce it locally. Keep the formatter dependency in the project and use the lock-backed installation path in CI. When upgrading it, review the new output and rule changes as a tooling change.

Pre-commit hooks can shorten the feedback loop, but they are a convenience. They can be skipped or behave differently across environments. CI remains the shared enforcement point. For a large repository, formatter caches or scoped local runs can improve feedback time, while the merge check should still cover the intended project source.

## Edge Cases and Trade-offs

Style can interact with syntax support. A formatter must understand the PHP versions and syntax the repository uses; an outdated tool may fail to parse newer constructs, while a newer rule set may emit syntax unavailable on the project's minimum PHP version. Configure the tool's target version where supported and test it against the full supported syntax range.

Some rules are contentious or have exceptions: multiline array layout, import grouping, declaration ordering, line length, and docblock formatting can all affect readability differently across codebases. A style guide is most useful when exceptions are few, explicit, and stable. A team should not continually override individual formatter choices in reviews after agreeing to the policy.

Some automated rules are marked risky because the fixer cannot guarantee that the result preserves behavior. Keep risky transformations disabled for routine formatting unless the team has reviewed the specific rules and their effects. A formatting command should not silently become a broad code migration.

## Testing

Formatting is usually verified by a check-mode command rather than by unit tests. However, when the initial formatter pass or a rule can alter executable constructs, syntax-check affected files and run the tests that protect their behavior. Test any custom formatter rule or plugin against representative source fixtures. A green formatter check only verifies source shape under that tool's rules; it does not validate output, data handling, or domain behavior.

## Common Mistakes

- Treating PSR-12 or a PER as automatically enforced by PHP.
- Assuming a formatter ruleset implements every sentence of the named style guide.
- Using different style configurations in local editors and CI.
- Letting formatter upgrades introduce an unreviewed repository-wide diff.
- Mixing mechanical formatting with a behavior change.
- Including vendor, generated, cache, or build files in the formatter's target paths.
- Allowing a formatter's risky transformations during routine style cleanup without reviewing them.
- Using a formatting check as a substitute for syntax checks, tests, or static analysis.
- Running auto-fix in CI and leaving the contributor to discover uncommitted changes after the job fails.

## Senior Engineer Thinking

A style tool should remove repeated decisions and produce a stable review signal. Start with a target the team understands, make its implementation reproducible, and enforce it at a boundary developers can predict. Keep the initial cleanup reviewable and treat formatter updates as changes to the policy implementation.

The aim is not to make every developer prefer the same typography. It is to ensure that source layout stops consuming attention that should go to behavior. Where readability and a mechanical rule conflict, discuss a narrow, documented policy exception rather than repeatedly hand-editing the formatter's output.

## Exercises

1. Select PSR-12 or a specific release of the Coding Style PER for a small PHP project. Record why it fits and which formatter version implements it.
2. Configure a PHP formatter to inspect `src/` and `tests/`, then add a non-mutating check to CI. Confirm that an intentionally misformatted file fails the check and that fix mode produces the expected diff.
3. Run the formatter over an existing project on a dedicated branch. Review a sample of files and identify any templates, generated files, or rules that need different treatment.
4. Compare the diagnostics from PHP-CS-Fixer and PHP_CodeSniffer for one style rule. Decide which tool owns enforcement and remove conflicting configuration.
5. Write a brief style-tool upgrade procedure that covers dependency update, changed output, tests, and CI.

## Review Questions

1. How do formatting, syntax checking, static analysis, tests, and automated refactoring differ?
2. Why should a style target and tool configuration be committed to the repository?
3. What is the practical difference between a formatter's fix mode and check mode?
4. Why might a style-only cleanup deserve its own change?
5. What can go wrong when formatter and CI versions differ?
6. Why should risky formatter rules receive separate review?
7. What does a green formatting check establish, and what does it leave unproven?
8. How should a team handle a style rule that consistently harms readability in one part of its codebase?

## Summary

Formatting is a team policy implemented by versioned tools and repository configuration. Choose a style target the project can support, define which files are in scope, and make the same non-mutating check available locally and in CI. Format existing code in a dedicated, reviewable change, and inspect tool upgrades as changes to the policy implementation.

A formatter makes source presentation consistent; it does not prove syntax, types, tests, security, or business behavior correct. Keep those responsibilities distinct, and reserve broad source transformations for the controlled workflow in [Chapter 103](103-automated-refactoring.md).

## References

- [PHP-FIG: PSR-12 Extended Coding Style Guide](https://www.php-fig.org/psr/psr-12/)
- [PHP-FIG: Coding Style PER](https://www.php-fig.org/per/coding-style/)
- [PHP-CS-Fixer documentation: configuration](https://github.com/PHP-CS-Fixer/PHP-CS-Fixer/blob/master/doc/config.rst)
- [PHP-CS-Fixer documentation: usage](https://github.com/PHP-CS-Fixer/PHP-CS-Fixer/blob/master/doc/usage.rst)
- [PHP_CodeSniffer documentation: usage](https://github.com/PHPCSStandards/PHP_CodeSniffer/wiki/Usage)
- [PHP Manual: `php -l`](https://www.php.net/manual/en/features.commandline.options.php)
