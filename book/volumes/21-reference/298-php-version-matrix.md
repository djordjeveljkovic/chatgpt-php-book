---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 298
title: PHP Version Matrix
slug: php-version-matrix
status: complete
summary: ../../_ai/chapter-summaries/298-php-version-matrix-summary.md
---

# Chapter 298 — PHP Version Matrix

## Why This Matters

A PHP answer is incomplete when it ignores the version and runtime in which the code executes. A feature may be legal in one supported version, deprecated in another, unavailable because an extension is missing, or operationally different between FPM and CLI. Version support is therefore a compatibility contract, not a number placed in `composer.json`.

## Four Different Version Questions

Separate these questions before making a claim:

| Question | Evidence | Typical failure |
| --- | --- | --- |
| What does the language permit? | PHP manual, migration guide, executable test | syntax or type error |
| What does the dependency graph permit? | `composer.json`, lock file, platform constraints | install or autoload failure |
| What does this artifact run with? | image, binary, extensions, SAPI, OPcache | works in CI, fails in production |
| What does the deployment fleet contain? | inventory and rollout telemetry | mixed-version behavior |

The PHP version reported by one shell is not proof of the FPM version, worker image, extension set, or deployed artifact.

## A Practical Matrix

Maintain a matrix for every supported environment. The exact version values belong to the project, but the dimensions should remain explicit.

| Dimension | FPM/web | CLI/cron | Queue worker | Build and test |
| --- | --- | --- | --- | --- |
| PHP binary/version | recorded | recorded | recorded | recorded |
| SAPI and extensions | recorded | recorded | recorded | recorded |
| `php.ini` and environment | recorded | recorded | recorded | recorded |
| Composer platform | artifact-controlled | artifact-controlled | artifact-controlled | pinned |
| OPcache/JIT policy | observed | usually different | usually different | not assumed |
| process lifetime | request | command | long-lived | test process |
| restart and rollout | pool reload | scheduler | graceful drain | reproducible |

Check the matrix during image build, startup, deployment, and incident diagnosis. A documented target without a runtime assertion is an aspiration.

## Language and Migration Boundaries

For each supported PHP release, classify changes as additions, behavior changes, deprecations, removals, or implementation details. Record the minimum version for syntax and APIs, but also record the minimum extension and SAPI assumptions. A migration plan should identify:

1. code that cannot parse on the oldest version;
2. behavior that remains syntactically valid but changes meaning;
3. deprecations that become production noise or future failures;
4. dependencies that raise the platform floor;
5. tests that exercise both old and new behavior;
6. the date or evidence required to remove compatibility code.

Do not infer a semantic guarantee from a release headline. Read the relevant migration guide and verify the behavior with a small test on every supported target.

## Composer and Extensions

Composer platform requirements express a dependency graph’s declared floor. They do not install a PHP binary, guarantee required extensions on every SAPI, or prove that production used the lock file. Validate the lock file, platform configuration, plugins, install scripts, generated autoload files, and artifact provenance.

Extensions deserve their own matrix: required, optional, development-only, and forbidden. Compare extension versions and configuration between FPM, CLI, workers, CI, and build images. A class existing in CLI is weak evidence if the request path uses a different SAPI.

## Mixed-Version Deployment

Assume old and new processes coexist during a rollout. Safe changes usually make the reader tolerant before the writer emits a new form:

1. deploy code that understands old and new inputs;
2. add compatible schema or message fields;
3. begin producing the new form;
4. observe old-reader traffic and drain old workers;
5. remove compatibility only after evidence shows it is unused.

The same rule applies to serialized jobs, cache values, database columns, HTTP responses, and feature-flag states. A rollback plan must include data and message compatibility; redeploying old PHP code alone may be unsafe.

## Verification Checklist

Ask:

- Which versions are supported, and until when?
- Which PHP binary and SAPI execute each path?
- Which extensions and ini settings are required?
- Does the lock file resolve under the target platform?
- Can old workers read new messages and schemas?
- What does OPcache retain during reload?
- Which deprecations are warnings today and failures tomorrow?
- What telemetry proves the fleet converged?
- What is the forward-recovery plan if rollback is unsafe?

## Common Mistakes

- treating `php -v` as the production inventory;
- testing only the newest PHP release;
- setting a Composer constraint without checking extensions;
- assuming FPM and CLI share configuration;
- emitting an incompatible message during a partial rollout;
- treating a lock file as proof of security or artifact provenance;
- deleting compatibility code without usage evidence;
- claiming implementation details are language guarantees.

## Exercises

1. Build a matrix for FPM, CLI, workers, CI, and the build image.
2. Design an expand-and-contract rollout for a new serialized job field.
3. Find three claims in a PHP upgrade proposal that need version or SAPI qualification.
4. Write startup assertions for PHP version, extensions, ini settings, and artifact identity.
5. Plan a deprecation-removal change with usage telemetry and a rollback boundary.

## Review Questions

- Why are language, dependency, artifact, and fleet versions different questions?
- Why can FPM and CLI disagree?
- What makes a mixed-version rollout safe?
- What does Composer not prove?
- When is rollback less safe than forward recovery?

## Summary

Version support is a matrix of language behavior, dependencies, extensions, SAPI configuration, artifacts, process lifetime, and fleet rollout. Make compatibility explicit, test every supported target, tolerate mixed versions deliberately, and remove old paths only with evidence.

## Chapter 299 Handoff

The next reference chapter turns recurring failures from this matrix and the earlier volumes into a diagnostic catalog: Common Mistakes.

## References

- [Chapter 93 — Dependency Resolution](../07-composer-and-the-php-ecosystem/093-dependency-resolution.md)
- [Chapter 270 — PHP 5 Codebases](../18-legacy-php/270-php-5-codebases.md)
- [Chapter 276 — Framework Migration](../18-legacy-php/276-framework-migration.md)
- [Chapter 288 — Code Review](../20-senior-engineering/288-code-review.md)
- [Chapter 297 — Senior PHP Interview Questions](../20-senior-engineering/297-senior-php-interview-questions.md)
