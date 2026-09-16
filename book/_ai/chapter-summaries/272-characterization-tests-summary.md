# AI Summary — Chapter 272 — Characterization Tests

- Status: complete
- Volume: Volume XVIII — LEGACY PHP
- Last updated: 2026-09-16

## Written material

Characterization tests are presented as observations of current behavior at a chosen legacy boundary, not automatic specifications of correctness. The chapter covers test charters, input/environment capture, observed versus required versus desired behavior, boundary selection, test oracles, a typed observation harness, fixture and state control, negative paths, external-effect recording, golden masters and snapshots, PHP 5/runtime limits, differential comparison, confidence, failure modes, exercises, and review questions.

## Concepts already explained

Characterization test, observation pipeline, test charter, observed behavior, required behavior, desired behavior, test oracle, normalization, side-effect record, fixture boundary, negative path, golden master, snapshot, differential comparison, difference classification, partial evidence, and characterization coverage.

## Terminology established

Entry-point boundary, environment input, observable output, effect identity, stable field, normalization rule, recording adapter, dry-run contract, provider sandbox, snapshot authority, runtime limitation, shared-state comparison, unknown behavior, confidence evidence, and removal decision.

## Examples used

The chapter includes an observation pipeline, observed/required/desired table, a test charter, typed `Observation` and `normalizeObservation()` examples, fixture requirements, negative-path cases, external-effect records, golden-master review steps, old/new differential comparison, coverage questions, failure modes, and characterization exercises.

## Cross-references

- [Chapter 158 — Why Tests Exist](../../volumes/11-testing/158-why-tests-exist.md)
- [Chapter 165 — Test Doubles](../../volumes/11-testing/165-test-doubles.md)
- [Chapter 170 — Test Design](../../volumes/11-testing/170-test-design.md)
- [Chapter 176 — Database Testing](../../volumes/11-testing/176-database-testing.md)
- [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../../volumes/16-distributed-systems/243-message-delivery.md)
- [Chapter 260 — Logging](../../volumes/17-production-engineering/260-logging.md)
- [Chapter 261 — Metrics](../../volumes/17-production-engineering/261-metrics.md)
- [Chapter 262 — Tracing](../../volumes/17-production-engineering/262-tracing.md)
- [Chapter 270 — PHP 5 Codebases](../../volumes/18-legacy-php/270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](../../volumes/18-legacy-php/271-legacy-architecture.md)
- [Chapter 273 — Safe Refactoring](../../volumes/18-legacy-php/273-safe-refactoring.md)

## Open threads

Continue Volume XVIII with Chapter 273 on safe refactoring, carrying forward narrow characterization boundaries, explicit oracles, controlled effects, and observed-versus-required distinctions.

## Exact next section

Chapter 273 — Safe Refactoring: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the PHP example, local Markdown links resolved, and `git diff --check` passed. PHPUnit and a PHP 5 runtime were not installed or run; the example was syntax-checked and the chapter records that modern lint cannot prove legacy-runtime compatibility. No live database, provider, queue, or shared-state comparison was run.

## Writing notes

Chapter 271 maps legacy architecture and ownership. Chapter 272 captures behavior at one boundary with explicit oracles and controlled effects. Keep behavior-preserving code transformations in Chapter 273 and detailed migration patterns in Chapters 274–278.
