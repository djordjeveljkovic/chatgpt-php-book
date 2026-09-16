---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 257
title: Containers
slug: containers
status: complete
summary: ../../_ai/chapter-summaries/257-containers-summary.md
---

# Chapter 257 — Containers

## Why This Matters

A container packages a process and its filesystem, environment, namespaces, and resource limits into a repeatable runtime unit. It improves delivery consistency and isolation, but it does not make an application stateless, secure, observable, or resilient automatically.

For PHP, a container may run Nginx, PHP-FPM, a queue worker, or a CLI job. Each role has a different lifecycle and resource profile. Understand the process boundary before choosing an image or orchestration setting.

## Image and Runtime Model

```text
image layers + configuration
          ↓ create
container namespaces and limits
          ↓ run
process → files, sockets, network, signals
```

An image is an artifact; a container is a runtime instance. Image layers are normally immutable, while the writable container layer is ephemeral. Durable data belongs in a database, object store, volume, or other explicit storage boundary.

Do not assume a container restart preserves files. A restart may create a new instance from the image. Temporary uploads, sessions, generated keys, queue state, and logs need deliberate destinations.

## Build a PHP Image

A conceptual multi-stage build separates dependency construction from runtime content:

```dockerfile
FROM composer:2 AS build
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist --no-interaction --no-progress
COPY . .

FROM php:fpm AS runtime
WORKDIR /app
COPY --from=build /app /app
USER www-data
CMD ["php-fpm", "-F"]
```

The tags, extensions, Composer invocation, and process user must be pinned and reviewed for the target environment. The example is illustrative, not a production-ready image. Build the application and dependencies in a controlled environment, scan and verify the artifact, and avoid copying development tools, VCS metadata, or secrets into the runtime image.

## One Role, One Lifecycle

A container may run more than one process, but a simpler operational model usually gives each container one primary role: web server, FPM, worker, scheduler, or migration command. If one process fails while another remains alive, an orchestrator may incorrectly consider the container healthy.

Run the foreground process when the runtime expects it. Handle termination signals, stop claiming new queue work, finish within a deadline, flush logs, and exit. A shell wrapper that does not forward signals can make graceful shutdown unreliable.

## Filesystem and Configuration

Treat the image as immutable. Mount only required writable paths and make their retention explicit. Do not write generated config or secrets into the image during a build.

Configuration can arrive as environment, mounted files, command arguments, or a platform-specific secret/configuration facility. Parse and validate it at startup. Environment values are strings and may contain unexpected whitespace, empty values, or false-looking text. See [Chapter 258 — Configuration](./258-configuration.md) and [Chapter 259 — Secrets](./259-secrets.md).

## Resource Limits

Containers can have CPU, memory, process, network, and filesystem limits. The application observes only part of the host and can be killed or throttled when the limit is reached. Size PHP-FPM workers, queue concurrency, OPcache, and temporary storage against container limits, not a larger host assumption.

```text
container memory limit
  > PHP worker peak × children
  + OPcache + FPM master + sidecars + shutdown margin
```

A CPU quota can make a process appear slow without high host-wide CPU. A process limit can prevent workers or subprocesses from starting. Record the effective limits in diagnostics.

## Health Checks

Liveness should tell the platform whether the process is running. Readiness should tell routing whether the role can accept useful work. A container health check that runs a migration or deep dependency query can create load and restart loops.

Keep checks cheap, bounded, and role-specific. A worker may be ready when it can reach its queue and has capacity; a web container may be ready when FPM and required configuration are available. Dependency checks should have a timeout and a policy for transient failure.

## Networking

Container names, service discovery, NAT, DNS, and network policy are runtime dependencies. A service name is not an authentication mechanism. Authenticate calls, validate certificates and host policy, and do not assume that a private network is trusted.

Connection pooling and DNS behavior can differ between containers and hosts. Configure finite connection and request timeouts and observe connection failures separately from application failures.

## Security

Use a minimal base image, pin or digest-lock important artifacts, run as a non-root user, drop unnecessary capabilities, use a read-only root filesystem when practical, and restrict network egress. Scan dependencies and image packages, but treat scanners as evidence rather than proof of safety.

Never put passwords, private keys, or long-lived tokens in an image layer or build argument that may be retained in build history. Rotate credentials and restrict runtime access. Keep a software bill of materials and provenance where the delivery policy requires it; see [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md).

## Deployment

Build once and promote the same immutable artifact through environments when possible. Record image digest, PHP version, extensions, Composer lock, configuration version, and migration state. Roll out gradually, observe readiness and business outcomes, and keep database and message contracts compatible across overlap.

Container restart is not recovery if the same bad configuration or dependency failure returns. Define restart limits, backoff, alerting, and operator actions. Avoid a restart loop that hides a persistent startup error.

## Testing

Build the image reproducibly, run PHP lint and application tests in it, verify required extensions and users, inspect the final filesystem for secrets and development tools, and exercise startup and shutdown. Test memory and CPU limits, unwritable paths, missing configuration, DNS failure, dependency timeout, signal forwarding, health checks, and artifact rollback.

## Common Mistakes

* Treating a container as durable storage.
* Copying secrets or development dependencies into the image.
* Running as root to avoid filesystem ownership work.
* Using a shell wrapper that swallows termination signals.
* Sizing workers from host memory instead of container limits.
* Making health checks perform expensive business work.
* Assuming a private container network is trusted.
* Restarting repeatedly without diagnosing a persistent failure.

## Senior Engineer Thinking

Ask what the container isolates, what it shares with the host and platform, which process owns its lifecycle, and where durable state lives. Containers are valuable when they make the runtime artifact and limits explicit; they are harmful when they hide process, signal, storage, and security behavior behind a deployment abstraction.

## Exercises

1. Design separate web, FPM, worker, and migration container roles and their health checks.
2. Inspect a PHP image for source, vendor, extensions, users, writable paths, and secrets.
3. Calculate a worker count from a container memory limit and measured PHP peak memory.
4. Test signal forwarding and queue drain during a container termination.

## Review Questions

* What is the difference between an image and a container?
* Which data must not live only in the writable container layer?
* Why should health checks be role-specific and bounded?
* How do container limits affect PHP-FPM sizing?
* Why is an internal container network not an authorization boundary?
* What does build-once deployment make easier to verify?

## Summary

Containers provide repeatable process artifacts, namespaces, and resource limits, but they do not supply durability, security, lifecycle correctness, or observability automatically. Build minimal immutable PHP images, run the correct foreground process as a non-root user, externalize state and secrets, size workers against effective limits, make health checks cheap, and test startup, signals, limits, networking, and rollback.

## References

- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 258 — Configuration](./258-configuration.md)
