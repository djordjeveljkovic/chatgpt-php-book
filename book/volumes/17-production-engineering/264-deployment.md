---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 264
title: Deployment
slug: deployment
status: complete
summary: ../../_ai/chapter-summaries/264-deployment-summary.md
---

# Chapter 264 — Deployment

## Why This Matters

Deployment is the controlled change of a running system. It is not merely copying PHP files to a server. A release changes executable code, dependencies, configuration, database expectations, process state, routes, queues, and sometimes the behavior of external clients.

A deployment can be locally correct and still damage production. The new PHP code may require a database column that old workers do not know about. A configuration value may be present in the image but absent in the runtime environment. A queue may contain messages written by the previous release. A health check may report ready before OPcache, migrations, or secrets are usable. A rollout that has no observation and stop condition can turn a small defect into a fleet-wide outage.

Design deployment as a sequence of reversible or recoverable transitions. Build an immutable artifact, verify it before promotion, make versions compatible during overlap, change traffic gradually, observe customer impact, and keep a clear decision about when to pause, roll forward, or roll back.

## Mental Model

A production deployment crosses several boundaries:

~~~text
source and dependencies
        ↓ build and verify
release artifact
        ↓ configure
runtime instance
        ↓ start and observe
healthy capacity
        ↓ route gradually
customer traffic
        ↓ measure and decide
continue | pause | repair | rollback
~~~

The release artifact should identify the source revision, dependency lock state, runtime version, build inputs, and configuration schema it expects. Runtime configuration and secrets are supplied at deployment time according to the environment policy; they should not be baked into a public image or source archive.

Deployment has two separate questions:

1. Can this artifact start and pass its local checks?
2. Can this version safely coexist with the versions already serving traffic?

Passing the first question does not answer the second. Most rollout risk appears during mixed-version operation, when old and new PHP-FPM workers, queue consumers, schemas, caches, and clients overlap.

## Release, Instance, and Traffic

Keep these terms separate:

* a release is an immutable versioned artifact and its metadata;
* an instance is a running process or container using a release;
* capacity is the amount of work those instances can safely accept;
* traffic is the work routed or claimed by those instances;
* a deployment is the controlled transition between release populations.

Changing a release symlink, container image, PHP-FPM pool, Nginx configuration, or queue worker command can affect different populations and lifetimes. Identify the owner and observation boundary for each. Chapter 257 covered immutable container artifacts; Chapters 256 and 263 covered process lifecycle and readiness.

Do not confuse “process started” with “release serving.” A process can start with the wrong configuration, fail to reach a required dependency, or serve only a subset of routes. Use startup and readiness checks appropriate to the role, then verify the externally visible path.

## Build an Immutable Artifact

A reproducible build should obtain dependencies from the lock file, run static and security checks, produce the intended generated files, and record enough metadata to identify what was deployed. For a PHP application, this often includes:

* a pinned PHP runtime and required extensions;
* composer.json and composer.lock;
* production dependency installation without development packages;
* generated autoload files;
* application source and compiled frontend assets where relevant;
* configuration schema and release metadata;
* checksums, provenance, and vulnerability results.

Build once and promote the same artifact through environments. Rebuilding separately for staging and production can produce different dependency archives, timestamps, generated code, or toolchain output. If environment-specific configuration changes the artifact itself, make that difference explicit and verify it separately.

Do not copy secrets into the artifact. A secret loaded at build time can remain in a layer, cache, generated file, error report, or dependency configuration. Chapter 259 described secret lifecycle and rotation; deployment must preserve that boundary.

## Compatibility During Rollout

Assume old and new versions overlap unless the deployment mechanism proves otherwise. Compatibility includes:

* HTTP request and response schemas;
* queue message schemas and acknowledgment behavior;
* database columns, indexes, constraints, and stored data;
* cache keys and serialized values;
* configuration names and defaults;
* feature flags and external provider contracts;
* PHP runtime and extension requirements.

Use additive changes first. A common database sequence is:

~~~text
expand: add nullable/new structure
    ↓
deploy code that can read old and new
    ↓
backfill or dual-write under observation
    ↓
switch reads or writes
    ↓
contract: remove old structure after all consumers migrate
~~~

Do not drop a column in the same step that deploys code no longer using it if old workers, reports, replicas, or rollback artifacts may still reference it. A rollback of code cannot restore a destructive schema change. Chapter 277 will examine database migration in more detail; the deployment contract must already account for mixed versions.

Queue compatibility deserves the same care. New producers may send fields old consumers ignore, but changing a required field or interpretation can poison a queue. Retain message version, tolerate the supported overlap, and deploy consumers before producers when the protocol requires it.

## A Typed Deployment Gate

A deployment controller should turn evidence into an explicit decision. Keep the policy separate from the mechanism that talks to Kubernetes, a load balancer, SSH, systemd, or another platform:

~~~php
<?php

declare(strict_types=1);

enum ReleaseDecision: string
{
    case Continue = 'continue';
    case Pause = 'pause';
    case Stop = 'stop';
}

final readonly class RolloutObservation
{
    public function __construct(
        public int $readyInstances,
        public int $targetInstances,
        public float $errorRatio,
        public float $p99LatencyMs,
        public bool $healthChecksFresh,
    ) {
        if ($readyInstances < 0 || $targetInstances < 1) {
            throw new InvalidArgumentException('Invalid instance counts');
        }

        if ($readyInstances > $targetInstances) {
            throw new InvalidArgumentException('Ready instances exceed target');
        }

        if (!is_finite($errorRatio) || $errorRatio < 0 || $errorRatio > 1) {
            throw new InvalidArgumentException('Invalid error ratio');
        }

        if (!is_finite($p99LatencyMs) || $p99LatencyMs < 0) {
            throw new InvalidArgumentException('Latency cannot be negative');
        }
    }
}

function decideRollout(
    RolloutObservation $observation,
    float $maximumErrorRatio,
    float $maximumP99LatencyMs,
): ReleaseDecision {
    if (
        !is_finite($maximumErrorRatio)
        || $maximumErrorRatio < 0
        || $maximumErrorRatio > 1
        || !is_finite($maximumP99LatencyMs)
        || $maximumP99LatencyMs < 0
    ) {
        throw new InvalidArgumentException('Invalid rollout thresholds');
    }

    if (!$observation->healthChecksFresh) {
        return ReleaseDecision::Pause;
    }

    if ($observation->readyInstances < $observation->targetInstances) {
        return ReleaseDecision::Pause;
    }

    if (
        $observation->errorRatio > $maximumErrorRatio
        || $observation->p99LatencyMs > $maximumP99LatencyMs
    ) {
        return ReleaseDecision::Stop;
    }

    return ReleaseDecision::Continue;
}
~~~

This is a policy example, not a complete rollout controller. A real controller needs observation windows, minimum sample sizes, rollback or repair actions, lease ownership, idempotent commands, and protection against acting on stale metrics. A deployment should pause when it cannot trust its health evidence; it should not interpret missing telemetry as a successful rollout.

## Deployment Strategies

### Recreate

Stop the old population, then start the new one. This is simple and can be appropriate for a singleton worker, a development environment, or a service whose contract tolerates downtime. It has an explicit availability cost and requires a safe drain or durable queue boundary.

### Rolling update

Replace instances incrementally while old and new versions overlap. This reduces the blast radius but requires mixed-version compatibility and enough spare capacity. A readiness check should gate traffic to new instances; a ready result does not mean the rollout is correct, so observe errors and latency after traffic begins.

### Blue-green

Run two complete populations and shift traffic from blue to green. This provides a clean traffic switch and can simplify rollback, but it doubles capacity requirements and does not remove schema or queue compatibility issues if both populations run simultaneously.

### Canary

Send a controlled fraction of traffic to the new release, compare it with a suitable baseline, and expand only when evidence meets the policy. Choose traffic that represents important routes, tenants, regions, and failure modes. A canary that receives only cheap GET requests cannot validate a checkout write path.

### Shadow or replay traffic

Send copied or synthetic requests to the new release without allowing side effects to escape. This can find performance and parsing problems, but a shadow request must not charge a card, send an email, mutate shared state, or access production data outside its policy. It is not a substitute for real canary traffic.

No strategy is automatically safer. Choose based on capacity, statefulness, compatibility, failure-domain isolation, rollback time, and the cost of an incorrect side effect.

## PHP-FPM and Nginx Deployment

For a release-directory layout, an application may use:

~~~text
/srv/app/releases/2026-09-16.3
/srv/app/releases/2026-09-16.4
/srv/app/current → /srv/app/releases/2026-09-16.4
~~~

The symlink change is only one step. Existing PHP-FPM workers may have already-compiled OPcache scripts from the previous release, while in-flight requests can continue executing the old code. With unique release paths, new requests resolve the selected path and compile it; with reused paths or timestamp validation disabled, explicitly apply the configured OPcache reset/reload policy. Verify that new requests execute the intended release. A worker recycle or pool reload should respect the drain and termination budgets from Chapter 256.

Nginx configuration should be syntax-checked before reload. Its master process can load the new configuration, start new workers, and gracefully shut down old workers; a failed configuration or socket change should leave the previous configuration serving when the platform supports that behavior. A reload is not a complete application deployment: verify the FastCGI target, active release, proxy trust, health route, and upstream timing.

Do not mutate the live code directory while workers are serving it. Partial file copies can produce mixed source, stale autoload maps, or syntax failures. Write the artifact to a new directory, verify it, atomically select the release, and manage process reload as a separate controlled step.

## Configuration, Secrets, and Feature Flags

A deployment should validate effective configuration before admitting traffic. Required values, runtime extensions, certificate paths, secret handles, and feature-flag defaults belong to a typed startup contract. The process should fail before serving if an immutable required input is missing.

Separate code rollout from risky behavior rollout when possible. Deploy code that can support a feature, keep the feature disabled, enable it for a bounded cohort, observe, and expand. A flag is not a substitute for compatibility: old code must tolerate the configuration and data states created while the flag changes.

Secret rotation can overlap deployment. A new release may need to verify both old and new key IDs or credentials during the transition. Do not revoke a credential merely because the new artifact has started; verify consumers, propagation, and rollback requirements first.

## Migrations and Background Workers

Run schema migrations under explicit ownership. Starting the same migration concurrently in every PHP-FPM container can create lock contention, duplicate work, or inconsistent readiness. A migration command should have a lock, timeout, progress evidence, and recovery policy appropriate to the database.

Deploy queue consumers with message compatibility before changing producers. Drain or pause a queue only when the delivery contract and business deadline permit it. A worker deployment must stop claiming new messages, finish current work, and acknowledge only after its durable effect succeeds. It must release or negatively acknowledge unfinished leases for redelivery, with idempotency protection for effects that may have completed before the interruption; then it flushes bounded telemetry and exits within its termination policy.

Scheduled jobs and CLI commands are part of the release population too. Record which release ran a job, make commands idempotent where retries are possible, and avoid running both old and new schedulers accidentally during overlap.

## Observability and Rollout Decisions

Observe the change at several levels:

* artifact and startup failures;
* ready capacity and drain state;
* request rate, error ratio, and latency distribution;
* dependency errors, queue age, and worker restarts;
* release, instance, region, and route population;
* business outcomes and reconciliation records.

Compare like with like. A new release receiving a different route mix or a traffic surge cannot be evaluated against an unqualified aggregate baseline. Use the trace and log correlation from Chapters 260–262 to investigate representative failures, while metrics determine prevalence and thresholds.

Define a stop condition before rollout. Examples include a sustained error-ratio increase, a p99 latency regression with a meaningful sample, readiness loss, queue age growth, unexpected restarts, or a business invariant violation. Define an observation window and a recovery condition too. Pausing a rollout without an owner or next action is merely a delayed incident.

## Failure Modes and Recovery

Common deployment failures include:

* artifact cannot start because a PHP extension or configuration value is missing;
* new workers fail readiness because a required secret or schema is unavailable;
* mixed versions disagree about a queue or cache representation;
* a migration locks a hot table and consumes database capacity;
* OPcache or release selection leaves some workers on old code;
* a new Nginx configuration routes requests to the wrong socket;
* a canary receives too little representative traffic;
* rollback code cannot read the new schema or old secrets have already been revoked;
* observability export fails, hiding a regression;
* a rollout controller acts repeatedly because its command is not idempotent.

Recovery is a deployment design requirement. Keep the last known-good artifact available, preserve compatible configuration and verification keys, record rollout state durably, and make each step safe to retry. Chapter 265 will focus on rollback; this chapter establishes the compatibility and evidence needed for rollback to be possible.

## Performance and Capacity

A rollout changes capacity. Rolling updates may temporarily add surge instances, while draining old instances remain alive and consume CPU, memory, connections, and licenses. Size the overlap against host, database, queue, and provider limits. A deployment that doubles PHP-FPM workers without increasing database capacity may reduce availability.

Warm-up can alter latency. New workers may pay OPcache, autoload, connection, cache, or JIT-related costs. Do not send full traffic immediately just because readiness answers quickly; use a warm-up policy when cold-path latency affects users.

Deployment speed is not the same as deployment safety. A fast rollout reduces the time spent in mixed-version state but can increase blast radius before evidence arrives. A slow rollout consumes overlap capacity and prolongs compatibility requirements. Choose a rate from failure detection time, rollback time, capacity headroom, and business risk.

## Security and Supply Chain

Authenticate artifact sources, pin dependencies and base images according to policy, verify checksums or signatures where supported, and record provenance. The deployment identity should have only the permissions needed to promote artifacts, update runtime configuration, change traffic, and observe rollout state.

Do not let untrusted pull requests publish to a production registry or change deployment credentials. Protect build logs, generated manifests, secret references, and rollback artifacts. A deployment system can be compromised even when the PHP application is secure.

Validate image and release contents before promotion. Do not rely on a runtime health check to detect embedded credentials, development tools, debug settings, writable code paths, or unexpected source files. Chapter 157 covered supply-chain controls; Chapters 257–259 covered artifact, configuration, and secret boundaries.

## Concurrency and Coordination

Only one controller should own a rollout for a given service and environment unless the system defines how controllers coordinate. Use a durable lease or compare-and-set revision to prevent two deploy jobs from shifting traffic in opposite directions.

Make actions idempotent. “Set desired release to X” is safer than “advance one step” when a command may be retried after an ambiguous response. Persist the observed revision, target revision, stage, timestamps, and decision evidence. A deployment controller that loses its connection must be able to resume without duplicating migrations, traffic shifts, or notifications.

Coordinate deployment with autoscaling. An autoscaler may add capacity while a rollout controller counts ready instances, and terminating instances may remain visible during the grace period. Use the platform's documented population fields and define whether the target is desired, updated, ready, or serving capacity.

## Testing

Test deployment as a workflow:

* build from a clean checkout using the intended lock file;
* verify artifact contents, runtime version, extensions, and provenance;
* start with missing and invalid configuration;
* exercise startup and readiness transitions;
* test old and new application versions against the pre-expand and expanded schemas when partial migration is possible;
* publish old and new queue messages, then consume each with old and new consumers through the supported overlap;
* run migration and deploy commands concurrently to verify ownership;
* verify Nginx syntax, FastCGI routing, OPcache/reload behavior, and release selection;
* inject startup failure, readiness loss, dependency timeout, crash, and exporter failure;
* pause at thresholds and confirm that the controller does not advance;
* retry every deployment action after an ambiguous response;
* drain workers and confirm no message or request is lost outside its contract;
* rehearse rollback with the actual previous artifact, configuration, and secrets.

Use a staging environment that preserves the relevant process, network, database, queue, and capacity boundaries. A unit test for decideRollout cannot prove that a real load balancer stops sending traffic after readiness changes. Test policy and mechanism separately, then exercise the integrated deployment path.

## Common Mistakes

* Rebuilding a supposedly identical artifact for each environment.
* Deploying code and a destructive schema change in one irreversible step.
* Assuming a process start or health response proves customer readiness.
* Ignoring old PHP-FPM workers, OPcache, queue messages, or cache values.
* Running migrations concurrently from every application instance.
* Rolling out faster than the time needed to observe a regression.
* Treating a canary with unrepresentative traffic as evidence for all users.
* Revoking old secrets before all consumers and rollback paths migrate.
* Allowing two deployment controllers to act on the same service.
* Making deployment actions non-idempotent.
* Treating missing metrics or traces as a successful rollout.
* Leaving debug settings, credentials, or writable code in the artifact.

## Senior Engineer Thinking

The senior question is not “did the deployment command succeed?” It is “which versions, schemas, processes, clients, and data states overlap, what evidence proves the new population is safe, and what recovery remains possible if that evidence is wrong?”

A deployment is a distributed transaction over code, configuration, state, traffic, and time. It rarely has one atomic commit. Reliability comes from compatibility, bounded transitions, durable coordination, health-aware routing, observation, and recovery practice.

Prefer a deployment design that makes a bad release small, visible, and recoverable. If rollback depends on a destructive schema change being undone instantly, an old secret remaining available, or an untested controller path, rollback is a hope rather than a capability.

## Exercises

1. Design a rolling deployment for a PHP-FPM service with three instances, one database migration, a queue consumer, and an optional feature flag. List the mixed-version compatibility requirements.
2. Extend the RolloutObservation policy with a minimum sample count, an observation window, and a pause condition for readiness loss.
3. Draw an expand-and-contract migration sequence for renaming a column while old and new workers overlap.
4. Rehearse a failed canary caused by a new dependency timeout. Identify the metrics, traces, logs, durable records, stop condition, and recovery action.

## Review Questions

* Why is a deployment more than copying PHP files?
* What is the difference between an artifact, an instance, capacity, traffic, and a deployment?
* Why must old and new versions be compatible during a rollout?
* Why are additive database changes safer than destructive changes during overlap?
* How do startup and readiness checks participate in deployment?
* What can go wrong when PHP-FPM workers, OPcache, or Nginx reload asynchronously?
* Why should migration ownership be separate from ordinary application startup?
* How do canary traffic and observation windows affect rollout safety?
* Why must deployment actions be idempotent?
* What evidence must remain available for rollback?

## Summary

Deployment is a controlled transition among release artifacts, runtime instances, capacity, traffic, configuration, schemas, queues, and external contracts. Build once, verify provenance and contents, promote the same artifact, preserve mixed-version compatibility, use startup/readiness gates, roll out according to capacity and failure risk, and define stop and recovery conditions before changing traffic. Treat PHP-FPM, OPcache, Nginx, migrations, secrets, queues, autoscaling, observability, and deployment coordination as separate boundaries. A successful command is not a safe release; safety comes from bounded transitions, representative evidence, idempotent control, and practiced recovery.

## References

- [Kubernetes: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes: Updating a Deployment](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [Nginx: Controlling nginx](https://nginx.org/en/docs/control.html)
- [PHP Manual: FastCGI Process Manager](https://www.php.net/manual/en/install.fpm.php)
- [Composer: install command](https://getcomposer.org/doc/03-cli.md#install)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 255 — Nginx](./255-nginx.md)
- [Chapter 256 — PHP-FPM](./256-php-fpm.md)
- [Chapter 257 — Containers](./257-containers.md)
- [Chapter 258 — Configuration](./258-configuration.md)
- [Chapter 259 — Secrets](./259-secrets.md)
- [Chapter 260 — Logging](./260-logging.md)
- [Chapter 261 — Metrics](./261-metrics.md)
- [Chapter 262 — Tracing](./262-tracing.md)
- [Chapter 263 — Health Checks](./263-health-checks.md)
- [Chapter 265 — Rollback](./265-rollback.md)
- [Chapter 277 — Database Migration](../18-legacy-php/277-database-migration.md)
