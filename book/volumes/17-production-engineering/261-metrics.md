---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 261
title: Metrics
slug: metrics
status: complete
summary: ../../_ai/chapter-summaries/261-metrics-summary.md
---

# Chapter 261 — Metrics

## Why This Matters

Logs explain individual events. Traces explain a path through several components. Metrics answer a different question: how often is something happening, how large is it, and is the system inside its expected operating envelope?

Without metrics, an incident becomes a collection of anecdotes. Someone reports that checkout “feels slow,” an operator finds a timeout in a log, and a developer guesses whether the problem is widespread. With well-designed metrics, the team can see request rate, error rate, latency distribution, queue age, worker saturation, and dependency pressure before choosing an intervention.

Metrics are measurements with a cost. Every name, label value, bucket, and retention period consumes memory, network, storage, and query budget. A metric that includes an unbounded user ID or URL can create millions of time series and become an outage of its own. Design a metric for a decision: what question will it answer, who will act on it, and what bounded dimensions make that answer useful?

## Mental Model

A metric is not just a number. It has at least these parts:

```text
measurement = name + value + time + bounded attributes + unit + meaning
```

For example:

```text
http.server.request.duration
value: 184
unit: milliseconds
attributes: route=/checkout, method=POST, status_class=2xx
meaning: one completed request took 184 ms
```

The same value can mean different things depending on its instrument. A counter records accumulated occurrences. A gauge describes a value that can rise and fall, such as current queue depth. A histogram records a distribution of observations, which allows a query to estimate percentiles and the proportion below a threshold. Do not compare or aggregate instruments merely because their names look similar.

Metrics are usually aggregated before or during storage. The application may emit individual observations, but an exporter or collector commonly turns them into rates, sums, counts, bucket populations, and time-windowed series. A dashboard therefore shows an interpretation of measurements, not an immutable truth independent of time range, aggregation, or sampling.

## Core Concept: Signals for Decisions

Start with the operational decision, then choose a signal. A checkout team may need to answer:

| Question | Useful signal | Typical bounded dimensions |
| --- | --- | --- |
| Are requests failing? | request counter and error counter | route, method, status class |
| Are users waiting? | duration histogram | route, method, outcome |
| Is capacity filling? | in-flight requests, worker utilization, queue age | pool, queue, service |
| Is a dependency unhealthy? | call count, error count, duration histogram | dependency, operation, result |
| Is work being delayed? | oldest message age and queue depth | queue, priority |

The dimensions are deliberately coarse. `route=/users/{id}` is useful; `route=/users/847291` is usually a cardinality mistake. `status_class=5xx` supports a broad failure view; a separate bounded `status=500` dimension may be useful when each status has an operationally distinct response. The right set depends on the decisions the team must make.

Use a small number of stable metric families. A request metric family might include:

* `http.server.request.count`, a counter of completed requests;
* `http.server.request.error.count`, a counter for server-side failures;
* `http.server.request.duration`, a histogram in milliseconds;
* `http.server.request.in_flight`, a gauge for current concurrency.

The exact names can follow an existing telemetry convention. Consistency across services is more valuable than inventing a clever local vocabulary.

## Counters, Gauges, and Histograms

A counter should represent an accumulated quantity that does not decrease during the life of the process. Process restarts can reset a local counter; a metrics backend must account for that when calculating a rate. The useful question is often not the total count but the change per unit of time:

```text
request rate = change in completed requests / elapsed time
error ratio  = change in errors / change in completed requests
```

Guard the denominator. A period with no requests has no meaningful error ratio, and a tiny sample can make a percentage look dramatic while affecting almost nobody.

A gauge represents a current observation. Queue depth, resident memory, active workers, and in-flight requests are gauges. A gauge is not automatically a counter just because its value is numeric. Setting a queue-depth gauge to zero after a failed read would be misleading if the queue was merely unobservable; distinguish zero from unknown when the collection path can fail.

A histogram preserves a distribution through counts and boundaries. Average duration alone hides tail behavior: 99 requests at 20 ms and one request at 2 seconds have a very different user experience from 100 requests at 40 ms, even when averages are close. Choose boundaries that reflect product and infrastructure thresholds, such as the response-time objectives for an endpoint. Do not choose hundreds of buckets without a query or alert that needs them.

## How It Works Across a PHP Request

The application has a local measurement boundary, but it is not the entire telemetry system:

```text
PHP request
    ↓ records observations
metric SDK or adapter
    ↓ batches/exports
collector or agent
    ↓ aggregates and transports
metrics backend
    ↓ queries rules and dashboards
operator decision
```

The PHP process may be short-lived under PHP-FPM. A process-local counter therefore describes one worker, not the whole service. Aggregation must happen across workers, hosts, containers, and releases. A long-running queue worker has a longer local lifetime, but it still must not be treated as the authoritative store for business counts: a crash can discard process-local state.

Collection failure is part of the design. If exporting blocks the request, telemetry can increase user-visible latency. If the application drops every observation immediately, an incident may become invisible. Prefer bounded buffers, bounded export time, and a stated drop policy. For a critical business fact such as “payment captured,” use durable domain state or an audit record; a metric is not a replacement for that evidence.

## A Small PHP Boundary

Keep application code independent from a particular collector. This boundary records the semantic operation and leaves batching, export, aggregation, and backend-specific representation to the adapter:

~~~php
<?php

declare(strict_types=1);

interface MetricSink
{
    /** @param array<string, bool|int|string> $attributes */
    public function increment(
        string $name,
        int $value = 1,
        array $attributes = [],
    ): void;

    /** @param array<string, bool|int|string> $attributes */
    public function record(
        string $name,
        int|float $value,
        array $attributes = [],
    ): void;

    /** @param array<string, bool|int|string> $attributes */
    public function add(
        string $name,
        int $value,
        array $attributes = [],
    ): void;

    /** @param array<string, bool|int|string> $attributes */
    public function set(
        string $name,
        int|float $value,
        array $attributes = [],
    ): void;
}

final readonly class RequestMetrics
{
    public function __construct(private MetricSink $metrics)
    {
    }

    public function completed(
        string $routeTemplate,
        string $method,
        int $statusCode,
        int $durationMs,
    ): void {
        $attributes = [
            'route' => $routeTemplate,
            'method' => $method,
            'status_class' => (string) (intdiv($statusCode, 100) * 100) . 'xx',
        ];

        $this->metrics->increment('http.server.request.count', 1, $attributes);
        $this->metrics->record(
            'http.server.request.duration',
            max(0, $durationMs),
            $attributes,
        );

        if ($statusCode >= 500) {
            $this->metrics->increment(
                'http.server.request.error.count',
                1,
                $attributes,
            );
        }
    }
}
~~~

The adapter should validate metric names and attribute keys, apply an allow-list or bounded schema, and impose limits on queued observations. The caller supplies a route template rather than a raw path because the template has bounded values. The wrapper also normalizes a negative duration to zero; a clock or measurement bug should be investigated separately rather than producing an invalid observation.

An in-flight gauge needs a paired increment and decrement around the request boundary. That operation must be exception-safe:

~~~php
<?php

declare(strict_types=1);

/** @param callable(): mixed $handler */
function measureRequest(
    MetricSink $metrics,
    string $routeTemplate,
    callable $handler,
): mixed {
    $attributes = ['route' => $routeTemplate];
    $metrics->add('http.server.request.in_flight', 1, $attributes);

    try {
        return $handler();
    } finally {
        $metrics->add('http.server.request.in_flight', -1, $attributes);
    }
}
~~~

This second example intentionally exposes an important limitation: an add/up-down operation is suitable for tracking changes, but only an adapter with defined cross-process semantics can provide a fleet-wide concurrent value. Two overlapping requests using independent process-local state still cannot update one shared gauge. In a real instrumentation adapter, use an instrument with defined add/up-down semantics or have the collector derive concurrency from reliable start and completion events. The code illustrates the boundary, not a universal implementation of cross-process concurrency.

## Practical Metric Design

Instrument boundaries where a decision or resource changes:

* at request admission and completion;
* around outbound HTTP and database calls;
* when a queue accepts, starts, retries, completes, or dead-letters work;
* when a worker starts, drains, recycles, or fails;
* when a cache serves, misses, refreshes, or evicts an entry.

Record both work and waiting. A queue with a stable depth can still be unhealthy if the oldest message is approaching its deadline. A dependency with a low error rate can still violate the user-facing latency objective if its slow tail is on every critical request. Pair volume, errors, duration, and saturation instead of building a dashboard from whichever number was easiest to emit.

Use exemplars or links to logs and traces when the telemetry platform supports them. A metric can show that the 99th percentile rose; a trace or structured log can show which dependency path produced the tail. Keep the link metadata bounded and safe. The metric itself should remain aggregatable.

## Production Example: A Queue Worker

Suppose a PHP worker consumes invoice messages. A useful set of observations is:

```text
queue.message.accepted         counter  queue=invoice
queue.message.started          counter  queue=invoice
queue.message.completed        counter  queue=invoice, outcome=success|failure
queue.message.retry.count      counter  queue=invoice, reason=timeout|temporary|conflict
queue.message.age              histogram  queue=invoice
queue.depth                    gauge  queue=invoice
queue.oldest_age               gauge  queue=invoice
worker.in_flight               gauge  pool=invoice
worker.restarts                counter  pool=invoice, reason=memory|signal|crash
```

Do not use the message ID, customer ID, exception text, or arbitrary retry reason as an unbounded label. If the business needs per-message history, store that in a durable message or operation record and link to it from safe logs or traces. The metric should answer “is invoice processing healthy?” rather than “what happened to message `abc...`?”

A queue alert should consider more than depth. A large queue can be normal during a scheduled import if age remains within the contract. A small queue can be unhealthy if consumers are stopped and new work has also stopped. Combine arrival rate, completion rate, oldest age, and worker availability with the service-level objective.

## Bad Example and Better Example

This metric appears convenient:

```text
checkout.failed{user_id="847291", url="/checkout?coupon=...", exception="..."} 1
```

It has several problems. The values are high-cardinality, the URL and exception can contain sensitive data, and the resulting series cannot be usefully aggregated. It also turns a metric backend into an accidental event store.

A bounded alternative is:

```text
checkout.request.error.count{
    route="/checkout",
    method="POST",
    failure="dependency_timeout",
    status_class="5xx"
} 1
```

Keep `failure` an enumerated category owned by the application boundary. If the category set grows without review, it is no longer bounded in practice. Put diagnostic details in a redacted structured log and retain the operation or trace ID only where that correlation is safe.

## Edge Cases

### Restarts and resets

A PHP-FPM worker restart resets process-local counters and gauges. A backend that computes rates must detect counter resets; a dashboard should not interpret a restart as a sudden drop in traffic. Record process or release restarts separately so a reset has context.

### Unknown is not zero

If a queue exporter cannot reach the broker, reporting queue depth as zero falsely says that no work exists. Emit an exporter error, mark the observation unavailable if the system supports that state, or alert on freshness of the measurement. A missing sample and a measured zero are different operational facts.

### Clock and duration boundaries

Measure elapsed time with a monotonic clock where the runtime provides one. Wall-clock adjustments can make durations negative or unexpectedly large. Use wall-clock timestamps for event correlation, not for subtracting request durations. Chapter 224 introduced the measurement boundary; Chapter 238 covered monotonic deadlines and timeout phases.

### Retries and duplicates

Count attempts and logical operations separately. A retrying HTTP client may produce five dependency calls for one user operation. Naming the two counts differently prevents a retry storm from being mistaken for user traffic. For a queue, distinguish message deliveries from successfully completed business effects.

### Partial collection

One host, worker, or exporter may disappear. Aggregate metrics should expose the population being observed: active instances, scrape freshness, or export errors. A “healthy average” over the remaining hosts can hide a failed fraction of the fleet.

## Performance and Capacity

Instrumentation has overhead in PHP: function calls, attribute construction, serialization, queueing, network export, and backend indexing. Measure that overhead on the hot path. Keep attributes small, avoid serializing complete request bodies, and batch exports outside the latency-critical operation when the reliability contract permits.

Cardinality is a capacity problem. If a metric family has (n) names, (d_1, d_2, \ldots, d_k) possible values for each dimension, and (r) active resources or replicas, a rough upper bound is:

```text
series ≈ n × d₁ × d₂ × … × dₖ × r
```

The actual backend may store fewer series because combinations do not occur, but design against plausible traffic, tenants, routes, and deployment overlap. A new label is a schema and capacity change. Review it like a database index or a queue limit.

Sampling can reduce cost, but it changes what a metric means. Sampling individual requests may be appropriate for traces or detailed logs; counters used for error budgets generally need a defined aggregation strategy. Never sample away the only signal for a rare but severe failure without documenting the trade-off.

## Security and Privacy

Metric labels are often copied into dashboards, alert notifications, long-term storage, and third-party systems. Do not put passwords, tokens, cookies, authorization headers, email addresses, raw IP addresses, full URLs, SQL text, exception messages, or user-controlled strings in labels by default. Even a hashed identifier can be sensitive and high-cardinality.

Use an allow-list of dimensions and normalize values at the boundary. Restrict who can query metrics if they reveal tenant names, internal topology, deployment details, or traffic patterns. Protect metric exporters and collector endpoints; an unauthenticated endpoint can disclose service behavior or become a resource-exhaustion target.

Metrics are not automatically anonymous because they are numeric. A count broken down by a small set of rare customer categories may still disclose business or personal information. Apply the same data classification and retention reasoning used for logs in Chapter 260.

## Database Interaction

Database metrics should describe resource and query behavior without turning SQL text into a label. Useful signals include query duration histograms by bounded operation name, transaction rollbacks, lock waits, deadlocks, connection-pool usage, and rows or bytes processed where the adapter can measure them safely.

Prefer a normalized query fingerprint or application operation such as `invoice.list_recent` over a raw query containing IDs and literal values. Keep database metrics distinguishable by role and outcome: primary versus replica, read versus write, timeout versus deadlock, and committed versus rolled back. Explain the query plan with database tooling and logs; do not assume a metric can replace `EXPLAIN`.

When a request makes ten queries, request duration alone does not prove that the database caused the delay. Correlate query count and query time with application spans or bounded operation metrics. This preserves the boundary between an application symptom and a database diagnosis.

## Concurrency

Metrics are collected while the system changes. In a single PHP-FPM request, local variables are isolated from other requests. Across workers and hosts, no PHP variable is a shared atomic counter. A static property or in-memory registry does not provide fleet-wide correctness.

Use a collector, shared atomic store, or backend aggregation mechanism whose concurrency semantics are explicit. Define whether a gauge is sampled, incremented, or derived. Protect paired observations such as queue start/completion and in-flight work against exception paths and worker shutdown. A worker killed between “started” and “completed” should be visible as unfinished work or a timeout, not silently converted into success.

## Testing

Test the metric contract at the application boundary:

* a successful request emits the expected counter, duration, and bounded attributes;
* a server failure increments the error family exactly once;
* client validation failures are classified according to the service policy;
* route templates are used instead of raw paths;
* retry attempts and logical operations are not conflated;
* unknown collection is not represented as zero;
* exceptions and early returns still close in-flight measurements;
* forbidden values never become attributes;
* metric names and failure categories remain stable.

Use a fake `MetricSink` to assert semantic calls, then separately test the adapter's batching, export timeout, drop policy, serialization, and backend mapping. A unit test cannot prove that a production collector aggregates multiple PHP-FPM workers correctly; that needs an integration or telemetry-contract test.

## Common Mistakes

* Tracking only averages and hiding latency tails.
* Adding raw user IDs, URLs, SQL, or exception text as labels.
* Treating a process-local PHP counter as a service-wide total.
* Reporting an unavailable dependency or queue as zero.
* Counting retry attempts as if they were logical operations.
* Alerting on a single noisy host without knowing the observed population.
* Blocking requests on unbounded metric export.
* Using metrics as durable business or audit evidence.
* Creating a new metric for every log event instead of choosing a stable aggregation.
* Adding labels without estimating time-series cardinality and retention cost.

## Senior Engineer Thinking

The senior question is not “what else can we instrument?” It is “which decision will this measurement improve, at what cost, and how will we know when the measurement itself is wrong?”

Good metrics form a contract between the application and the people operating it. Names, units, dimensions, reset behavior, freshness, aggregation, and retention all matter. A dashboard can be visually polished and operationally useless if it omits denominator, tail latency, saturation, or collection health.

Use metrics to detect and quantify a condition; use logs, traces, durable records, and controlled experiments to explain and change it. The strongest production systems make those signals correlate without making any one signal responsible for every question.

## Exercises

1. Define request, dependency, and queue metrics for a PHP checkout service. For each metric, state its instrument, unit, dimensions, and operator decision.
2. Review a metric schema containing `user_id`, raw URL, exception text, and tenant name. Replace unsafe or unbounded dimensions with a bounded design and explain where diagnostic detail belongs.
3. Design alerts for error ratio, p99 latency, queue age, and worker saturation. Include denominator, evaluation window, missing-data behavior, and a recovery condition.
4. Implement a fake `MetricSink` and test that a request failure records one error, one completion, and one duration observation even when the handler throws.

## Review Questions

* How does a counter differ from a gauge and a histogram?
* Why is an average insufficient for latency?
* Why are raw URLs and user IDs dangerous metric dimensions?
* What is the difference between a measured zero and an unavailable measurement?
* Why must retry attempts and logical operations have separate metrics?
* Why cannot a PHP-FPM worker's in-memory counter represent the whole service?
* Which evidence belongs in durable business state rather than a metric?
* How should metric collection failure affect the request path?

## Summary

Metrics are bounded quantitative signals for operational decisions. Choose counters, gauges, and histograms according to meaning; measure volume, errors, latency distributions, waiting, and saturation; aggregate across PHP-FPM workers and deployment units; and distinguish unknown from zero. Keep labels stable, small, safe, and bounded. Treat cardinality, export overhead, resets, retries, partial collection, and retention as capacity concerns. Metrics reveal that a condition exists; logs, traces, durable records, and tests provide the evidence needed to understand and safely change it.

## References

- [PHP Manual: `hrtime()`](https://www.php.net/manual/en/function.hrtime.php)
- [OpenTelemetry: Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [OpenTelemetry: Metric semantic conventions](https://opentelemetry.io/docs/specs/semconv/)
- [Chapter 224 — Measuring Performance](../15-performance/224-measuring-performance.md)
- [Chapter 238 — Timeouts](../16-distributed-systems/238-timeouts.md)
- [Chapter 260 — Logging](./260-logging.md)
- [Chapter 262 — Tracing](./262-tracing.md)
