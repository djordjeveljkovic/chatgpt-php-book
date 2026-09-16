---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 266
title: CI/CD
slug: ci-cd
status: complete
summary: ../../_ai/chapter-summaries/266-ci-cd-summary.md
---

# Chapter 266 — CI/CD

## Why This Matters

Continuous integration and continuous delivery are the control system between a change and a running release. CI validates a change in a repeatable environment. CD promotes a verified artifact through environments and, when authorized, changes production. The names are less important than the boundaries.

A pipeline can be green and still be unsafe. It may test a different PHP version, resolve dependencies again instead of using the lock file, omit a queue worker, run untrusted code with a publishing credential, or deploy an artifact different from the one tested. A deployment approval can also be valid while the service is already in an incident or the rollback window has closed.

Design the pipeline as evidence-producing automation. Every stage should have a declared input, a bounded output, a failure result, an owner, and a clear authority to promote. The pipeline should make a safe change easy to verify and an unsafe change difficult to publish.

## Mental Model

Separate source, verification, artifact, promotion, and activation:

~~~text
change
  ↓
inspect and verify
  ↓
build immutable artifact
  ↓
test the artifact
  ↓
attest and publish
  ↓
promote the same artifact
  ↓
deploy with authorization
  ↓
observe and decide
~~~

CI is not “whatever runs on a pull request.” CD is not “a shell script after merge.” Both are workflows with trust boundaries. A test runner may execute application code; an artifact publisher may write to a registry; a deployment job may change production traffic. Give each only the permissions it needs.

## Continuous Integration and Continuous Delivery

### Integration

Integration combines changes and checks that the resulting system remains compatible. Useful checks include formatting, static analysis, unit tests, database integration tests, contract tests, security scanning, dependency policy, and a production-shaped build.

Integration does not require every test on every change. It requires that the selected gate be honest about its scope. A fast unit suite can gate a pure policy; it cannot prove a real database lock, PHP-FPM signal, queue lease, or provider response.

### Delivery

Delivery makes a verified artifact available for release. It may deploy automatically to a lower environment and require explicit authorization for production. The same bytes should move through the promotion path. Rebuilding for production creates a new artifact and invalidates evidence from the earlier test.

### Deployment

Deployment activates a release in an environment. Chapter 264 owns that deployment execution: configuration, secrets, schema compatibility, traffic, workers, health gates, and observation. Chapter 265 owns rollback and recovery decisions. A pipeline invokes those controls; it does not make deployment safe by itself.

### Continuous deployment

Continuous deployment automatically promotes every change that satisfies its policy. This can reduce batch size and manual delay, but only when tests, ownership, rollback readiness, and production signals are strong enough. Automation should not be used to hide an absent decision policy.

## Pipeline Stages

A useful pipeline makes the transition between stages explicit:

| Stage | Input | Evidence | Typical authority |
| --- | --- | --- | --- |
| Source | commit or merge result | immutable revision | repository policy |
| Fast checks | source tree | formatting, static analysis, unit results | automated gate |
| Integration | source plus services | database, queue, HTTP, and contract results | automated gate |
| Build | verified source and lock file | artifact digest and manifest | build identity |
| Artifact tests | immutable artifact | runtime and production-shaped results | automated gate |
| Publish | verified artifact | registry record and provenance | publishing identity |
| Promote | published artifact | environment approval and compatibility | release owner |
| Deploy | artifact plus runtime inputs | rollout and health evidence | deployment identity |
| Observe | serving release | customer and business signals | operator/controller |

Do not let an informal “green” label replace the evidence. Store test results with the commit, tool versions, PHP version, extensions, database engine, artifact digest, and environment assumptions that made them meaningful.

## Source and Change Identity

Start from an immutable source identity: a commit ID or equivalent content-addressed revision. A branch name can move. A pull request can be rebased. A mutable tag can be retargeted. Record the exact source revision in build metadata and deployment records.

Review risk at the change boundary. Changes to workflow files, build scripts, Dockerfiles, Composer configuration, dependency locks, deployment manifests, authorization policy, and migration code deserve stronger review because they can alter the pipeline itself.

A pipeline should distinguish:

* proposed source, which may be untrusted;
* reviewed source, which satisfies merge policy;
* built source, which ran in a controlled build;
* released artifact, which passed artifact verification;
* deployed revision, which was activated in an environment.

These identities must be linked, not inferred from a branch name or a human message.

## PHP Build Reproducibility

For a PHP application, the build should normally:

* select the supported PHP runtime and required extensions;
* install dependencies from composer.lock with the intended production options;
* reject an invalid or stale lock file;
* generate autoload files and required frontend or cache assets;
* run static analysis and security policy checks against the intended source scope;
* produce a clean artifact without development secrets or writable source paths;
* record PHP, Composer, extension, operating-system, and build-tool versions;
* calculate an artifact digest and retain the manifest.

Use the same build inputs for verification and publication. A second Composer install from the same valid lock file should select the locked versions, but changed lock files, install options, platform versions, Composer scripts, generated outputs, or other build inputs can still produce a different result. Cache immutable downloads for speed, but validate the resulting lock-backed artifact rather than treating a cache hit as proof.

## A Typed Gate Result

Keep gate policy separate from CI-provider APIs. A result should say what was checked and whether the evidence is usable:

~~~php
<?php

declare(strict_types=1);

enum GateDecision: string
{
    case Pass = 'pass';
    case Fail = 'fail';
    case Unknown = 'unknown';
}

final readonly class GateResult
{
    public function __construct(
        public string $gate,
        public GateDecision $decision,
        public int $passed,
        public int $failed,
        public string $artifactDigest,
    ) {
        if ($gate === '') {
            throw new InvalidArgumentException('Gate name cannot be empty');
        }

        if ($artifactDigest === '') {
            throw new InvalidArgumentException('Artifact digest cannot be empty');
        }

        if ($passed < 0 || $failed < 0) {
            throw new InvalidArgumentException('Gate counts cannot be negative');
        }

        if ($decision === GateDecision::Pass && $failed > 0) {
            throw new InvalidArgumentException('A passing gate cannot have failures');
        }
    }
}

function canPromote(GateResult ...$results): GateDecision
{
    if ($results === []) {
        return GateDecision::Unknown;
    }

    $artifactDigest = null;

    foreach ($results as $result) {
        if ($artifactDigest !== null && $artifactDigest !== $result->artifactDigest) {
            return GateDecision::Unknown;
        }

        $artifactDigest ??= $result->artifactDigest;

        if ($result->decision === GateDecision::Unknown) {
            return GateDecision::Unknown;
        }

        if ($result->decision === GateDecision::Fail) {
            return GateDecision::Fail;
        }
    }

    return GateDecision::Pass;
}
~~~

The example prevents results for different artifact digests from being combined into one authorization. A real gate also needs a test-run identity, expiry, required gate set, environment policy, approval record, and protection against result tampering.

## Test Scope and Ordering

Order checks by feedback speed and failure isolation:

1. parse configuration and validate repository structure;
2. format and statically analyze changed source;
3. run focused unit and policy tests;
4. run broader unit and feature tests;
5. build the production artifact;
6. run artifact-level smoke and runtime checks;
7. exercise real database, queue, HTTP, and framework boundaries;
8. run contract, security, and migration compatibility checks;
9. publish only after the required gates pass.

The order is a cost and diagnostic policy, not a correctness shortcut. A fast failure should identify the first violated invariant. Keep test fixtures bounded, isolate external resources, preserve failure logs, and record the command and environment.

Run a compatibility matrix when the release changes a boundary:

* supported PHP versions and extensions;
* old and new application versions against both pre-migration and expanded schemas when partial migration is possible;
* every supported old/new producer–consumer pairing for changed message contracts;
* current and rotated secret keys;
* framework and database versions that the product supports;
* feature flag disabled, cohort, and full-exposure states.

Do not call a matrix “complete” merely because every job is green. State which combinations are intentionally unsupported.

## Artifact Promotion

Promotion moves an already verified artifact between environments. It should preserve artifact digest, source revision, dependency lock, manifest, provenance, test evidence, and approval history. Environment-specific inputs may change, but that difference must be explicit and validated.

Use a release record that answers:

~~~text
what source produced this artifact?
which dependency and runtime versions were used?
which gates passed, and against which artifact?
who approved promotion?
which configuration and secret references will be used?
what rollback target remains available?
~~~

Never put secret values in a release record. Store references, key IDs, versions, and verification outcomes according to the secret policy from Chapter 259.

## Environments and Promotion Gates

An environment is more than a name. It includes data classification, network access, dependencies, capacity, secrets, traffic, and change authority. “Staging passed” is weak evidence if staging has no queue, uses a different database engine, or bypasses the production proxy.

Use promotion gates appropriate to the risk:

* required review for high-impact changes;
* dependency and vulnerability policy;
* migration compatibility and estimated lock duration;
* artifact provenance and signature verification;
* current incident or maintenance state;
* rollback target and capacity check;
* canary observation and business-invariant checks;
* explicit expiry for approvals and stale test results.

An approval should bind to the artifact and environment. An approval for commit X should not authorize commit Y merely because both have the same branch name.

## Deployment Concurrency

Prevent conflicting pipeline runs from publishing or deploying at the same boundary. Two commits may be tested concurrently, but production activation needs an ordering rule. Options include queueing, canceling obsolete pending runs, or allowing only a release controller with compare-and-set state.

Do not cancel a running deployment without knowing its cleanup semantics. A canceled job may leave a migration, traffic shift, lock, temporary environment, or partially published artifact. Record cancellation state and let a recovery owner reconcile it.

Use a concurrency key scoped to the service, environment, and operation class. A global lock can serialize unrelated work; a missing lock can let two controllers change traffic in opposite directions.

## Secrets and Untrusted Code

Treat pull-request source, issue text, branch names, filenames, generated test data, and dependency scripts as potentially hostile inputs. Composer scripts execute project code, so dependency installation is also an execution boundary. Do not interpolate untrusted values into shell code. Keep untrusted test jobs away from production credentials and publishing permissions.

Use short-lived workload identity where the platform supports it. Separate identities for testing, artifact publication, promotion, and deployment. A test runner should not be able to promote its own unreviewed output merely because it produced a green result.

Pin third-party actions or build components according to the organization’s supply-chain policy. Review workflow permission changes as production-code changes. Protect workflow files and deployment configuration with ownership and required review.

## Caching and Parallelism

Caching reduces feedback time but creates trust and invalidation boundaries. Key caches by relevant inputs: PHP version, extension set, operating system, lock-file hash, tool version, and build mode. Do not restore writable dependency or build caches created by untrusted pull-request jobs into privileged jobs; use isolated namespaces and verify outputs. A cache should accelerate a reproducible operation, not replace it.

Parallel jobs consume finite runner, database, queue, registry, and provider capacity. An unconstrained test matrix can make the system under test fail from load generated by CI. Bound concurrency, partition fixtures, isolate tenants, and tag tests by required services.

Do not share mutable database state between parallel jobs without an isolation design. Do not share a writable Composer vendor tree between jobs that can execute different lock files. Make cleanup safe to retry and report leaked resources.

## Observability and Delivery Feedback

Emit structured pipeline events for source acceptance, gate start and completion, artifact creation, publication, promotion, deployment, approval, cancellation, and failure. Correlate them with the release, artifact, environment, service, and incident identities.

Pipeline duration is not the only feedback signal. Track queue delay, execution time, retry count, flaky-test rate, cache hit rate, artifact publication failures, deployment lead time, change failure rate, recovery time, and escaped defects. Interpret these as engineering signals, not vanity targets.

An unavailable test service is not the same as a passing test. Mark infrastructure failure and unknown evidence separately. If a required gate cannot run, choose whether policy pauses, fails, or permits an explicitly documented exception with an owner and expiry.

## Failure Modes and Recovery

Common pipeline failures include:

* tests pass against one dependency graph while another graph is published;
* a cached artifact contains output from a different lock file or PHP version;
* a migration test passes on SQLite but fails under the production database engine;
* a pull request can execute shell injection in a privileged workflow;
* an artifact is published without a link to source and test evidence;
* a deployment job starts from a stale approval or old health signal;
* two pipeline runs deploy different revisions concurrently;
* cancellation leaves a lock, environment, or traffic change behind;
* a flaky test is retried until green and hides a real regression;
* a registry or runner outage is mistaken for an application failure;
* a green pipeline cannot roll back because the previous artifact was garbage-collected.

Recovery starts with preserving the run, logs, artifact references, permissions, and current deployment state. Stop promotion when evidence is incomplete. Reconcile partially created artifacts and environments. Use Chapter 265’s rollback and containment decisions when the pipeline has already changed production.

## Performance and Capacity

Optimize for useful feedback, not the smallest wall-clock number. Measure time spent waiting for runners, dependencies, containers, databases, approval, artifact storage, and deployment observation separately.

Run cheap deterministic checks early and expensive boundary tests in parallel only within capacity. Reuse immutable artifacts and dependency downloads safely. Split a broad suite by stable scope, but preserve a scheduled full path that catches interactions the fast path omits.

A pipeline can overload production-like services. Bound browser sessions, database connections, queue consumers, provider calls, and artifact downloads. Reserve capacity for retries and failure diagnosis. A ten-minute pipeline that consumes an hour of operator recovery is not fast.

## Security and Supply Chain

The pipeline is a production control plane. Protect it with least-privilege identities, isolated runners where needed, reviewed workflow changes, immutable artifacts, provenance, dependency policy, secret minimization, and auditable approvals.

Avoid long-lived cloud credentials in generic jobs. Do not expose secrets to jobs that execute untrusted code. Masking is not a guarantee that every transformed value remains hidden; prevent secrets from entering logs, command arguments, artifacts, caches, and test output.

Artifact attestations and signatures provide evidence about origin or integrity, but they do not prove that the application is correct. Combine them with tests, review, compatibility checks, and deployment observation. Chapter 157 covers the broader supply-chain boundary.

## Testing the Pipeline Itself

Test the pipeline as software:

* validate workflow and deployment configuration syntax;
* run each required gate with a deliberately failing fixture;
* verify that a failed or unknown gate cannot publish;
* confirm gate results bind to the exact artifact digest;
* test cache hits, misses, stale keys, and poisoned entries;
* exercise untrusted input through branch names, filenames, and generated output;
* revoke a test credential and verify least-privilege failure;
* interrupt artifact publication, promotion, and deployment at each stage;
* retry commands after ambiguous responses and verify convergence;
* run two competing deployments and verify the concurrency policy;
* expire an approval and confirm that it cannot promote;
* preserve enough logs and identifiers to diagnose a failed run;
* rehearse recovery after a partial production deployment.

The pipeline should have a small test fixture for each security and state transition, not only a large end-to-end workflow that fails opaquely.

## Common Mistakes

* Rebuilding artifacts after the tests that supposedly verified them.
* Treating branch names, tags, or approval comments as artifact identity.
* Running every test in one serial job with no failure boundaries.
* Running a large parallel matrix against one shared mutable database.
* Retrying flaky tests until the pipeline is green without recording instability.
* Giving pull-request code publishing or production permissions.
* Storing credentials in logs, caches, test reports, or generated manifests.
* Treating an unavailable service as a successful gate.
* Using a staging environment that omits the production boundary under test.
* Letting canceled jobs leave migrations, locks, traffic, or environments behind.
* Deploying from a mutable tag without verifying its digest.
* Deleting the previous artifact before the rollback window expires.

## Senior Engineer Thinking

The senior question is not “how quickly can the pipeline turn green?” It is “what evidence binds this exact change to this exact artifact, which authority may promote it, what boundaries were actually tested, and what happens when automation is incomplete or wrong?”

A mature CI/CD system makes the safe path observable and repeatable. It does not claim certainty beyond its test scope. It keeps source, artifact, environment, approval, deployment, and recovery identities connected, while keeping untrusted code and high-impact credentials apart.

## Exercises

1. Design a PHP pipeline that tests source and the immutable artifact separately. List every version and input that must be recorded.
2. Extend GateResult with a test-run identity, artifact digest, expiry, and required-gate policy. Define behavior for stale or missing evidence.
3. Model two concurrent production deployments and choose a queue, cancellation, or compare-and-set policy. Include cleanup after cancellation.
4. Threat-model a pull-request workflow that runs Composer scripts and publishes a container. Identify untrusted inputs, credentials, artifacts, and approval boundaries.

## Review Questions

* What is the difference between CI, delivery, deployment, and continuous deployment?
* Why must tests and publication bind to the same immutable artifact?
* Which evidence belongs in a release record?
* Why is a green unit suite not proof of database, queue, or production safety?
* How should unknown or unavailable gate results affect promotion?
* Why can a cache improve speed without proving reproducibility?
* Which pipeline identities should be separated?
* How can cancellation leave production state that still needs recovery?
* Why should approvals bind to an artifact and environment rather than a branch?
* What does it mean to test the pipeline itself?

## Summary

CI/CD is evidence-producing automation across source, verification, artifact creation, promotion, deployment, and observation. Bind every gate and approval to exact source and artifact identities, build once and promote the same immutable result, test the boundaries that matter, separate untrusted code from high-impact credentials, bound concurrency and cache trust, and treat unknown or partial pipeline states as recovery work. A fast green pipeline is valuable only when its evidence is scoped, durable, authorized, and connected to the release that reaches users.

## References

- [GitHub Actions: Security for GitHub Actions](https://docs.github.com/en/actions/how-tos/secure-your-work)
- [GitHub Actions: Deployment environments](https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments)
- [GitHub Actions: Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
- [GitHub Actions: Workflow syntax and actions reference](https://docs.github.com/en/actions/reference/workflows-and-actions)
- [SLSA: Supply-chain Levels for Software Artifacts](https://slsa.dev/spec/v1.2/)
- [Composer: install command](https://getcomposer.org/doc/03-cli.md#install)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 164 — Contract Tests](../11-testing/164-contract-tests.md)
- [Chapter 176 — Database Testing](../11-testing/176-database-testing.md)
- [Chapter 235 — Scaling](../15-performance/235-scaling.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 257 — Containers](./257-containers.md)
- [Chapter 258 — Configuration](./258-configuration.md)
- [Chapter 259 — Secrets](./259-secrets.md)
- [Chapter 260 — Logging](./260-logging.md)
- [Chapter 261 — Metrics](./261-metrics.md)
- [Chapter 262 — Tracing](./262-tracing.md)
- [Chapter 264 — Deployment](./264-deployment.md)
- [Chapter 265 — Rollback](./265-rollback.md)
- [Chapter 267 — Backups](./267-backups.md)
- [Chapter 277 — Database Migration](../18-legacy-php/277-database-migration.md)
