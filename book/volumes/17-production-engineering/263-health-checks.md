---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 263
title: Health Checks
slug: health-checks
status: complete
summary: ../../_ai/chapter-summaries/263-health-checks-summary.md
---

# Chapter 263 — Health Checks

## Why This Matters

A health check is a control signal between a running application and the system responsible for routing, restarting, or alerting on it. A good check answers one narrow question: can this process start, should it receive this class of traffic, or is it still alive enough to recover?

An endpoint that returns “OK” while every PHP-FPM child is blocked on a database is not useful readiness evidence. An endpoint that queries the database on every liveness probe can make an outage worse. A check that exposes connection strings, exception messages, or internal topology creates a security problem while trying to provide operational information.

Health is scoped. A process can be alive but not ready for checkout. A service can be ready for cached reads while its optional recommendation provider is unavailable. A deployment can be started but not yet warmed. Separate these states, define their consequences, and make probe behavior bounded and safe.

## Mental Model

Health checks sit in a control loop:

~~~text
probe caller
    ↓ request
health endpoint or status source
    ↓ bounded observation
health result
    ↓ routing, restart, alert, or operator decision
system state changes
    ↺ probe repeats
~~~

The caller might be a load balancer, an orchestrator, a service-discovery system, a deployment controller, or an operator. The caller's reaction matters as much as the response body. A readiness failure should usually remove an instance from new traffic; a liveness failure may trigger a restart; a startup failure may keep later probes from acting too aggressively during initialization.

Health checks do not prove correctness of every request. They are coarse sensors with limited reach. They should be cheap enough to run repeatedly, specific enough to drive one action, and honest about uncertainty.

## Startup, Liveness, and Readiness

Use distinct checks for distinct decisions.

### Startup

Startup asks whether initialization has completed. It can cover configuration parsing, required extensions, dependency wiring, migrations or cache warm-up when those are part of the process contract, and the point at which the process can evaluate liveness and readiness safely.

A slow-starting process needs a startup budget. Without one, a liveness probe may restart a healthy process before it finishes loading configuration or warming a required cache. A startup check should eventually fail decisively when initialization cannot complete; otherwise a broken deployment can remain in an indefinite starting state.

### Liveness

Liveness asks whether the process is capable of making progress or needs replacement. Keep it shallow. A PHP-FPM liveness path might verify that the front controller or FPM ping path can execute a bounded response and that the process has not entered a known unrecoverable state. It should not require a healthy database or payment provider unless the process cannot recover without that dependency.

Restarting a process does not repair an external outage. A liveness check that fails because the database is down can cause every instance to restart together, increasing connection storms and delaying recovery. Use liveness for process failure, not general service dissatisfaction.

### Readiness

Readiness asks whether the instance should receive a particular class of work now. It can include required configuration, local capacity, a connection pool, a required database, schema compatibility, queue consumer state, or a bounded dependency policy. A readiness failure should normally stop new traffic while allowing the process to remain alive and recover.

Readiness is not necessarily binary across all operations. One instance may be ready for public reads but not for writes, or ready for interactive requests but not bulk jobs. Use separate roles or explicit capability checks when the routing system supports them. Do not turn one broad “healthy” bit into an undocumented policy for every caller.

## The Check Contract

Define the check's owner, input, timeout, dependency scope, result, and reaction:

| Check | Question | Typical reaction | Dependency depth |
| --- | --- | --- | --- |
| Startup | Has required initialization completed? | delay later probes or fail deployment | local and required startup inputs |
| Liveness | Can this process make progress? | restart or replace instance | shallow local observation |
| Readiness | Should new work be routed here? | remove from traffic | required path and capacity |
| Deep diagnostic | What is failing? | alert or operator investigation | broader, authenticated, slower |

The same endpoint should not silently change meaning based on who calls it. If a platform needs a cheap probe and operators need a deep report, expose separate paths or an explicit authenticated mode with different limits.

Return a status that matches the caller's contract. Many HTTP probe systems treat a 2xx response as success and a 5xx response as failure, but exact handling of redirects, timeouts, and body content belongs to the caller's documentation. Do not assume that a JSON field saying healthy overrides an HTTP failure status.

## What PHP Does at the Boundary

PHP executes the health handler as ordinary application code. It can inspect configuration, local files, process state exposed by an extension or sidecar, and dependencies through clients. It cannot infer that another PHP-FPM worker is healthy merely because the current worker answered.

Under PHP-FPM, a probe consumes a real worker. A slow or recursive health handler can exhaust the same pool needed to serve customers. Keep the code path short, avoid unbounded allocation, and set a timeout for every remote check. Route health traffic deliberately so a public endpoint does not accidentally reveal the FPM status page or bypass authentication rules.

The web server and FPM layer can also report different facts. Nginx may be reachable while the FastCGI socket is unavailable. FPM may accept a ping while the application cannot load a required configuration value. Name the layer being tested and compose results without pretending that one check covers the entire request path. Chapters 254–256 established the process, Nginx, and FPM boundaries.

## A Small PHP Boundary

Keep health checks behind a narrow interface. The endpoint decides policy; individual checks report bounded facts:

~~~php
<?php

declare(strict_types=1);

enum HealthState: string
{
    case Healthy = 'healthy';
    case Degraded = 'degraded';
    case Unhealthy = 'unhealthy';
    case Unknown = 'unknown';
}

final readonly class HealthResult
{
    public function __construct(
        public string $name,
        public HealthState $state,
        public int $durationMs,
    ) {
        if ($name === '') {
            throw new InvalidArgumentException('Health check name is required');
        }

        if ($durationMs < 0) {
            throw new InvalidArgumentException('Duration cannot be negative');
        }
    }
}

interface HealthCheck
{
    public function name(): string;

    public function check(int $timeoutMs): HealthResult;
}

final class ReadinessPolicy
{
    /** @param list<HealthResult> $results */
    public function statusCode(array $results): int
    {
        foreach ($results as $result) {
            if ($result->state === HealthState::Unhealthy) {
                return 503;
            }
        }

        return 200;
    }
}
~~~

This result deliberately does not include an exception message, hostname, DSN, or raw dependency response. A separate authenticated diagnostic report can retain more detail under a stricter policy. The Unknown state is useful when a check could not obtain a trustworthy observation; whether it fails readiness depends on the dependency's role and the service contract.

The timeout is an input to the check, not a promise that PHP can interrupt every blocking extension call at exactly that duration. The adapter must use the client library's documented timeout and enforce an outer budget where possible. A check that ignores the timeout argument is violating its contract.

## Practical Example: Required and Optional Dependencies

Suppose a storefront requires the catalog database for product pages but treats recommendations as optional. Model the policy explicitly:

~~~php
<?php

declare(strict_types=1);

final readonly class DependencyPolicy
{
    /** @param list<string> $required */
    public function __construct(public array $required)
    {
    }

    /** @param list<HealthResult> $results */
    public function allowsTraffic(array $results): bool
    {
        foreach ($results as $result) {
            if (
                in_array($result->name, $this->required, true)
                && $result->state !== HealthState::Healthy
            ) {
                return false;
            }
        }

        return true;
    }
}

function responseForStorefront(
    DependencyPolicy $policy,
    HealthResult ...$results,
): int {
    return $policy->allowsTraffic($results) ? 200 : 503;
}
~~~

The storefront can remain ready when recommendations are degraded, but only if the request path really has a safe fallback. The fallback must be authorized, bounded, and visible in metrics or logs. Marking every optional dependency healthy just to keep traffic flowing hides a product change and makes operators misread the state.

Do not reuse a generic dependency policy for all endpoints. A checkout write may require a primary database and payment authorization; a catalog read may use a bounded stale replica; a health endpoint itself may need only local process evidence.

## Production Example: Probe Composition

A production setup often composes layers:

~~~text
/health/startup
    └── configuration loaded, required extensions, initialization complete

/health/live
    └── process can execute a bounded local response

/health/ready
    ├── startup complete
    ├── instance is accepting work
    ├── required dependency policy
    └── local capacity and drain state

/health/diagnostic
    ├── authenticated operator access
    ├── dependency details
    ├── recent failure categories
    └── bounded evidence references
~~~

Keep the probe paths outside ordinary business middleware when that middleware can fail for reasons the probe is intended to classify. At the same time, do not bypass security middleware accidentally. Use a dedicated route with explicit authentication, rate limits, and access controls.

During graceful shutdown, mark readiness false before stopping new work. Continue liveness long enough for the process to drain if the platform's lifecycle expects that. A process that is intentionally draining is not dead; routing and restart policy should distinguish those states.

## Dependency Checks Without Amplification

A readiness request can become a fan-out request. If a load balancer probes 50 instances every five seconds and each probe calls three dependencies, the health system creates 30 dependency calls per second before customer traffic is counted. During an outage, those checks can synchronize and compete with recovery.

Use one or more of these controls:

* keep liveness local and dependency-free;
* cache a short-lived readiness observation when a small freshness window is acceptable;
* use a sidecar or agent for infrastructure checks;
* apply bounded parallelism and per-check deadlines;
* stagger probe schedules where the platform permits it;
* fail closed for a required invariant and degrade explicitly for optional work;
* expose dependency freshness and check errors as metrics.

Caching a readiness result introduces staleness. Define its maximum age and behavior when the refresh fails. A stale “ready” result may be acceptable for a few seconds during a rolling deployment, but not for a permission or financial invariant. Health policy is another consistency contract.

Do not use a database write as a health check merely to prove that writes work. It creates side effects, locks, replication traffic, and cleanup requirements. A bounded read or a dedicated dependency check is usually safer; an actual write test belongs in controlled integration or deployment verification.

## Edge Cases

### Startup migrations

If every replica runs a long migration while also reporting ready, traffic can reach incompatible schema states. If every replica reports not ready until a migration finishes, capacity can disappear. Separate migration ownership from application startup and make mixed-version compatibility explicit, as described in Chapter 264.

### Draining and deployment

A draining instance should fail readiness before it stops accepting work. If the router has a propagation delay, wait for the documented drain window before terminating the process. A health endpoint cannot make an upstream router forget an instance instantly.

### Dependency partitions

A timeout is not the same as a confirmed dependency failure. If a readiness check cannot obtain a result before its deadline, classify the result as unknown and apply the service's policy. Do not make a remote timeout hold an FPM worker indefinitely.

### Noisy neighbors

A shared host, database pool, or connection limit can make one instance unhealthy while another remains healthy. Include instance and dependency evidence in metrics and logs, but keep the public probe response bounded. Route or isolate workloads when the failure domain requires it.

### Probe traffic

Probe requests have real cost. Exclude them from customer request counts when that is the operational convention, or give them a stable route and explicit traffic class. Do not let probes distort latency objectives or trigger business side effects such as session creation, audit records, emails, or queue messages.

## Performance and Capacity

Health checks consume the same CPU, memory, FPM children, sockets, and dependency connections as other work unless they are isolated. A health endpoint that allocates a large report or performs sequential remote calls can become a capacity leak.

Set a total probe budget smaller than the caller's timeout and smaller than the time at which the platform must make a routing decision. Run independent checks in bounded parallelism only when the runtime and clients support it safely. Measure the check itself with the metrics and traces from Chapters 261–262, but avoid recursive instrumentation that makes the health path depend on a failing exporter.

Keep response bodies small. A machine needs a status code and perhaps a bounded version or state field; operators need a separate diagnostic surface. Do not return a list of every worker, URL, exception, or dependency response on every probe.

Health checks should be tested under probe concurrency, not just one request. Calculate the additional request and dependency rate from the number of callers, instances, interval, and fan-out. Apply the same finite-capacity reasoning used for FPM pools and backpressure.

## Security and Privacy

Public health responses should reveal the minimum necessary state. “ready” or “not ready” may be sufficient for a router. Detailed dependency names, hostnames, version strings, stack traces, environment values, and queue contents can help an attacker map the system.

Protect diagnostic endpoints with network policy and authentication. Restrict FPM status paths and never expose a FastCGI socket to an untrusted network; the PHP manual warns that an exposed FPM connection can allow request configuration that leads to code execution. Health checks must not become a bypass around application authorization.

Treat probe headers and trace context as untrusted input. Do not log arbitrary headers, reflect them in the response, or use them to choose a privileged diagnostic mode. If a check includes tenant-specific state, isolate that diagnostic operation from the global readiness result and apply tenant access controls.

## Database Interaction

A database readiness check should test the capability required by the traffic class, not merely whether a TCP socket accepts connections. Consider authentication, selected schema version, transaction mode, replica freshness, connection-pool availability, and the query's expected lock and resource cost.

Use a bounded, read-only query or a database-native readiness signal where appropriate. Do not run an expensive report query from the probe. A successful connection can still be misleading if the application cannot obtain a pool slot or if the required table/schema version is unavailable.

For a replica-backed read service, readiness may require a freshness bound. A replica that answers quickly with data older than the product contract is not ready for that read path, though it might still be suitable for a stale-tolerant endpoint. Express that distinction in the policy rather than a global database-up bit.

## Concurrency and Failure

Several probe callers can run concurrently. Protect shared caches, avoid a thundering herd on dependency refresh, and ensure a timeout or exception releases every local permit. A single slow health request should not block all later probes.

Health state can flap near a threshold. Use consecutive failures, recovery thresholds, minimum observation windows, or caller-side hysteresis when appropriate. Do not add long sleeps inside the endpoint; that consumes workers and makes the signal less timely. The controller should own restart and routing hysteresis.

A process can answer liveness while readiness is false for an extended period. That may be correct during a dependency outage, but alert on sustained readiness loss and capacity impact. Conversely, an instance that never answers liveness needs replacement even if its last readiness result was healthy.

## Testing

Test each health contract separately:

* startup remains false until required initialization completes;
* liveness does not depend on an unavailable optional provider;
* readiness fails for each required dependency failure category;
* optional dependency degradation produces the documented fallback;
* unknown and timeout results follow explicit policy;
* every dependency check respects its timeout and releases resources;
* concurrent probes do not create an unbounded dependency fan-out;
* draining sets readiness false before termination;
* malformed or unauthorized diagnostic requests reveal no internal details;
* probe traffic does not create business side effects;
* the HTTP status code agrees with the response body and caller contract.

Use fake checks and a fake clock for policy tests. Use integration tests for the actual FPM/Nginx route, dependency timeout behavior, status-page restrictions, orchestrator probe configuration, and graceful-drain sequence. Test a failure of the health exporter or metrics path separately; the application should still return its bounded health result according to policy.

## Common Mistakes

* Using one “health” endpoint for startup, liveness, readiness, and diagnostics.
* Making liveness depend on every external service.
* Running expensive or state-changing database queries from a probe.
* Returning HTTP 200 with a body that says the service is unhealthy.
* Exposing exception messages, hostnames, versions, or environment values publicly.
* Letting a probe fan out without a budget or cache.
* Ignoring FPM worker consumption and probe concurrency.
* Reporting stale or unknown data as a fresh healthy result.
* Leaving readiness true during drain or incompatible deployment.
* Restarting every instance because one shared dependency is unavailable.
* Forgetting that an optional fallback changes product behavior.

## Senior Engineer Thinking

The senior question is not “does the endpoint return OK?” It is “what decision will consume this signal, what failure does that decision repair, and what harm occurs if the signal is stale, noisy, slow, or wrong?”

Health checks are part of the control plane. They can remove capacity, restart processes, amplify dependency load, expose topology, and influence deployment safety. Give each check a named owner, bounded cost, explicit dependency policy, security boundary, freshness expectation, and tested reaction.

The best health design is often intentionally modest. Liveness protects against local process failure. Readiness protects traffic from an instance that cannot honor its contract. Startup protects initialization. Deep diagnostics help an authenticated operator explain the state. Metrics, traces, logs, and durable records then provide the evidence needed to decide what to repair.

## Exercises

1. Design separate startup, liveness, readiness, and diagnostic checks for a PHP-FPM checkout service. State each check's dependencies, timeout, status mapping, and caller reaction.
2. Calculate the dependency load generated by 100 instances, three probe callers, a five-second interval, and two dependency checks per readiness request. Add a cache or sidecar policy that keeps the load within a stated budget.
3. Implement fake HealthCheck objects and test required versus optional dependency policy, unknown results, timeouts, and graceful drain.
4. Review a public health response containing a DSN, exception message, hostname, and worker count. Redesign the public and authenticated diagnostic surfaces.

## Review Questions

* How do startup, liveness, readiness, and deep diagnostic checks differ?
* Why should liveness usually avoid external dependencies?
* Why can a health endpoint exhaust PHP-FPM capacity?
* What does an unknown dependency result mean for readiness?
* How can probe fan-out amplify an outage?
* Why should readiness change before graceful shutdown?
* What information belongs in an authenticated diagnostic endpoint rather than a public probe?
* How should a replica freshness requirement affect readiness?
* Why must the HTTP status code and response body agree?

## Summary

Health checks are bounded control signals for startup, liveness, readiness, routing, restart, and diagnosis. Separate their questions and reactions. Keep liveness shallow, make readiness capability-specific, give startup enough time, and expose deeper evidence only through protected diagnostics. Bound dependency fan-out, timeouts, caching, worker use, response size, and side effects. Distinguish unhealthy, degraded, unknown, draining, and not-yet-started states. Test probe concurrency, failure recovery, security, deployment transitions, and the actual Nginx/PHP-FPM boundary.

## References

- [Kubernetes: Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- [PHP Manual: FPM status page](https://www.php.net/manual/en/fpm.status.php)
- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: FastCGI Process Manager security](https://www.php.net/manual/en/install.fpm.php)
- [Nginx: upstream health checks](https://nginx.org/en/docs/http/ngx_http_upstream_hc_module.html)
- [Chapter 232 — PHP-FPM](../15-performance/232-php-fpm.md)
- [Chapter 246 — Backpressure](../16-distributed-systems/246-backpressure.md)
- [Chapter 254 — Linux for PHP Engineers](./254-linux-for-php-engineers.md)
- [Chapter 255 — Nginx](./255-nginx.md)
- [Chapter 256 — PHP-FPM](./256-php-fpm.md)
- [Chapter 260 — Logging](./260-logging.md)
- [Chapter 261 — Metrics](./261-metrics.md)
- [Chapter 262 — Tracing](./262-tracing.md)
- [Chapter 264 — Deployment](./264-deployment.md)
