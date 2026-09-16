# AI Summary — Chapter 101 — Static Analysis

- Status: complete
- Volume: Volume 7 — COMPOSER AND THE PHP ECOSYSTEM
- Last updated: 2026-09-15

## Written material

Explains static-analysis feedback and its limits, PHP native runtime declarations, PHPDoc generics and array shapes, analyzer stubs, false positives/negatives, gradual adoption and baseline debt, CI configuration, tool upgrades, testing boundaries, operational cost, exercises, and review questions. Formatting and automated refactoring remain in Chapters 102–103.

## Concepts already explained

- A static analyzer builds a model from source, native declarations, PHPDoc, stubs, and configuration; a clean report only covers the selected files and rules under those assumptions.
- PHP runtime type declarations enforce stated contracts at specific runtime boundaries, while a native `array` type does not validate its element shape.
- Generic and shaped-array PHPDoc enrich analyzers' models, but do not validate runtime values; external input requires runtime validation.
- Stubs model or correct external type information and must remain aligned with dependency behavior.
- False positives and false negatives arise from analysis approximations, missing dynamic behavior, exclusions, suppressions, or inaccurate annotations.
- Baselines enable incremental adoption but record debt; CI should prevent unexplained growth and analyze the intended source scope.
- Static analysis complements syntax checks, tests, security review, formatting, and refactoring tools.

## Terminology established

Static analysis, analyzer model, native declaration, PHPDoc, generic type, array shape, stub file, baseline, false positive, false negative, runtime validation, gradual adoption, diagnostic suppression, analyzer scope, result cache.

## Examples used

- `extractEmails()` with a `list<array{id: int, email: string}>` shape annotation.
- Generic `firstOrNull()` preserving the input list's element type in PHPDoc.
- Project-local PHPStan and Psalm CLI examples.
- A bad `@var` assertion versus validating data, adding a narrow stub, or locally suppressing an explained tool limitation.

## Cross-references

- [Chapter 94 — `composer.json`](../../volumes/07-composer-and-the-php-ecosystem/094-composer-json.md)
- [Chapter 95 — `composer.lock`](../../volumes/07-composer-and-the-php-ecosystem/095-composer-lock.md)
- [Chapter 102 — Formatting](../../volumes/07-composer-and-the-php-ecosystem/102-formatting.md)
- [Chapter 103 — Automated Refactoring](../../volumes/07-composer-and-the-php-ecosystem/103-automated-refactoring.md)
- PHP Manual, PHPStan, and Psalm primary documentation are cited in the chapter.

## Open threads

No chapter-specific open threads. Recheck analyzer-specific commands and PHPDoc behaviors against official docs before making tool-version-specific updates.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

PHP runtime declaration and coercion claims were checked against the PHP Manual. PHPStan/Psalm PHPDoc generics, array shapes, stubs, baselines, and CLI examples were checked against their current official documentation on 2026-09-15. All local links resolved, both PHP examples passed lint and runtime execution, and `git diff --check` passed. PHPStan and Psalm binaries are not installed in this workspace, so analyzer execution was not available. Independent proofreading found no necessary corrections.

## Writing notes

Keep formatting policy in Chapter 102 and automated transformations in Chapter 103. Do not present analyzer reports or PHPDoc as runtime validation or a proof of application correctness.
