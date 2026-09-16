# AI Summary — Chapter 174 — Flaky Tests

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Flaky tests fail without a relevant code change. The chapter covers nondeterministic time and randomness, condition waits, isolation, parallelism, diagnostics, retries, quarantine ownership, CI behavior, and production regression tests.

## Concepts already explained

Flaky test, source of nondeterminism, fake clock, monotonic clock, condition wait, first-failure evidence, quarantine, and flake rate.

## Examples used

A PHP fake clock and a bounded condition-wait helper using a monotonic timer.

## Cross-references

- [Chapter 163 — End-to-End Tests](../../volumes/11-testing/163-end-to-end-tests.md)
- [Chapter 172 — Mutation Testing](../../volumes/11-testing/172-mutation-testing.md)
- [Chapter 175 — Testing Legacy Code](../../volumes/11-testing/175-testing-legacy-code.md)

## Open threads

Continue with Chapter 175 on characterization and seams for legacy code.

## Exact next section

Chapter 175 — Testing Legacy Code: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; CI retries and browser infrastructure were not run.
