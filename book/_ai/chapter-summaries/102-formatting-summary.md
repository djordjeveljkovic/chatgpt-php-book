# AI Summary — Chapter 102 — Formatting

- Status: complete
- Volume: Volume 7 — COMPOSER AND THE PHP ECOSYSTEM
- Last updated: 2026-09-16

## Written material

Explains style-policy selection, PSR-12 and the evolving Coding Style PER, formatter/checker roles, PHP-CS-Fixer and PHP_CodeSniffer, repository configuration, local and CI checks, introducing style to legacy code, editor integration, risky rules, trade-offs, exercises, and review questions.

## Concepts already explained

- Formatting is a team policy enforced by versioned tooling; PHP does not enforce a project's source style.
- Syntax checks, formatters, static analyzers, tests, and refactoring tools answer different questions.
- PSR-12 is accepted; the Coding Style PER evolves. Tool ruleset coverage and status must be checked against the selected tool version.
- Reproducible configuration scopes owned PHP files, uses the same check locally and in CI, and keeps vendor/generated/build paths excluded.
- Initial formatting cleanups and formatter upgrades should remain reviewable; risky rules require deliberate review.

## Terminology established

Style target, formatter, style checker, fix mode, check mode, formatter ruleset, risky rule, formatting baseline, repository configuration.

## Examples used

- PHP-CS-Fixer configuration using the PER-CS ruleset and selected source/test paths.
- `.editorconfig` whitespace policy and Composer scripts for mutating and non-mutating formatter commands.
- Separate style-only cleanup and CI enforcement workflow.

## Cross-references

- [Chapter 94 — `composer.json`](../../volumes/07-composer-and-the-php-ecosystem/094-composer-json.md)
- [Chapter 95 — `composer.lock`](../../volumes/07-composer-and-the-php-ecosystem/095-composer-lock.md)
- [Chapter 100 — PSR Standards](../../volumes/07-composer-and-the-php-ecosystem/100-psr-standards.md)
- [Chapter 101 — Static Analysis](../../volumes/07-composer-and-the-php-ecosystem/101-static-analysis.md)
- [Chapter 103 — Automated Refactoring](../../volumes/07-composer-and-the-php-ecosystem/103-automated-refactoring.md)

## Open threads

No chapter-specific open threads. Check current formatter versions and PER status when updating version-sensitive examples.

## Exact next section

Chapter 103 — Automated Refactoring: the Why This Matters section.

## Technical verification notes

Style target and PHP-CS-Fixer configuration/usage claims were checked against PHP-FIG and PHP-CS-Fixer primary documentation on 2026-09-15. PHP_CodeSniffer behavior was checked against its project documentation. PHP 8.5.10 linted the PHP-CS-Fixer configuration example, the Composer script JSON parsed, 49 local links across the chapters and handoff files resolved, and `git diff --check` passed. Exact CLI behavior is version-sensitive and is called out in the chapter.

## Writing notes

Keep formatting policy separate from static-analysis correctness and semantic migrations. Chapter 103 owns automated source transformations.
