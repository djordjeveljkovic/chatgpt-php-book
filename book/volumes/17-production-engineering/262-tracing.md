---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 262
title: Tracing
slug: tracing
status: complete
summary: ../../_ai/chapter-summaries/262-tracing-summary.md
---

# Chapter 262 — Tracing

## Why This Matters

Metrics can tell you that checkout latency increased. Logs can show that one provider timed out. A trace can connect those facts into one causal path: the request entered Nginx, waited for a PHP-FPM child, executed application code, queried the database, called billing twice, and returned an error after the remaining deadline expired.

Tracing is especially valuable when a request crosses boundaries. A PHP application may call a database, cache, payment provider, and queue. Each component can be healthy in isolation while their composition is slow or partially failed. Without a shared trace context, an operator has to guess which log lines belong to the same operation and whether a queue message came from the request being investigated.

A trace is evidence, not a complete replay of execution. Sampling may omit requests, instrumentation may miss a library, clocks may differ between machines, and export may fail. Design traces to answer bounded diagnostic questions, then correlate them with metrics, structured logs, durable operation records, and tests.

## Mental Model

A trace is a directed collection of spans describing one operation or causal workflow:

~~~text
trace
└── server: POST /checkout
    ├── internal: validate request
    ├── client: database query
    ├── client: billing.charge
    │   └── client: billing retry attempt
    └── producer: invoice.created
~~~

A span has a name, kind, start and end time, attributes, events, status, and relationships to other spans. The parent relationship gives the usual tree shape. Links connect causally related spans when one operation has several parents or when preserving one parent would create a misleading tree.

The trace ID identifies the diagnostic workflow. A span ID identifies one timed component within it. Neither ID is an authorization credential. A trace context carried through an HTTP header or message envelope is metadata for correlation and sampling decisions; it must be parsed and validated, but it must not grant access.

## Spans and Boundaries

Create spans around meaningful boundaries:

* an inbound HTTP request;
* an outbound HTTP or database call;
* a queue publish, message delivery, or batch;
* a cache operation when its behavior explains the result;
* a significant application operation or state transition.

Do not create a span for every getter, array lookup, or trivial private method. That adds overhead and noise without exposing a boundary an operator can act on. A function-level span may be justified for a measured hot path or a long-running computation, but name that choice as an investigation or a stable operation rather than instrumenting the entire call graph reflexively.

Span kinds communicate the boundary's role. A server span receives a request. A client span makes an outbound request. A producer span sends a message. A consumer span processes one. An internal span represents work inside the service. Use the convention supported by the chosen telemetry system and keep the meaning stable across services.

Span names should be low-cardinality. POST /checkout or billing.charge is useful. GET /users/847291 or a full SQL statement with literal values is not. Put bounded attributes on the span and send sensitive or unbounded diagnostic detail to a separately governed evidence path, if it is needed at all.

## Trace Context Propagation

Context propagation carries the trace and span identifiers across a boundary:

~~~text
inbound carrier
    ↓ extract and validate
server span
    ↓ make current
application work
    ↓ inject into outbound carrier
child client/producer span
~~~

For HTTP, a standard propagation format such as W3C Trace Context can carry a traceparent and optional tracestate. The receiving service should validate the format, reject malformed values, and create a new root when no usable context is present. It should not blindly copy arbitrary headers into logs or downstream requests.

For a queue, put the propagation carrier in message metadata, not in the business payload when the broker provides a metadata area. A consumer extracts it and creates a consumer span with the producer context as its parent when that accurately describes the workflow. Delayed work may outlive the request by hours; the consumer must establish and clear its own current context for each message.

Batch consumers and fan-in operations often have several causal inputs. A single parent would imply that all work came from one message. Use span links or a bounded batch representation where the tracing system supports them, and keep the message IDs in durable records rather than high-cardinality span attributes.

Propagation is not cancellation. A child service may know the parent's deadline from a separate protocol field, but a trace header alone does not stop work. Propagation is not authentication either; authenticate the request or message independently.

## How It Works in PHP

A typical PHP request has these boundaries:

~~~text
Nginx or gateway
    ↓
PHP-FPM server span
    ↓
controller/application span
    ├── database client span
    ├── cache client span
    └── provider client span
          ↓
        provider server span, if instrumented
~~~

Under PHP-FPM, ordinary userland request state is created for one request and discarded when that request ends, although the FPM child process may serve many requests. The current span must therefore be attached to a request-scoped context and cleared at the boundary. A long-running worker has a different lifetime: a context, span, exception, or attribute buffer can leak from one message to the next unless the worker closes and resets it in finally.

The OpenTelemetry PHP API/SDK can provide these boundaries, and instrumentation libraries can cover frameworks or extensions. Keep application and library code dependent on the narrow API contract where possible; let the application composition root select the SDK, exporter, sampling policy, and resource metadata. Exact package names, extension support, and configuration options are version-sensitive and belong to the installed version's documentation.

## A Small PHP Boundary

The following interface illustrates the lifecycle a framework adapter must preserve without binding the chapter to a particular tracing package:

~~~php
<?php

declare(strict_types=1);

interface Span
{
    public function setAttribute(string $name, bool|int|float|string $value): void;

    public function recordException(Throwable $exception): void;

    public function setStatus(string $status): void;

    public function end(): void;
}

interface Tracer
{
    public function start(string $name): Span;
}

/** @param callable(Span): mixed $operation */
function inSpan(Tracer $tracer, string $name, callable $operation): mixed
{
    $span = $tracer->start($name);

    try {
        $result = $operation($span);
        $span->setStatus('ok');

        return $result;
    } catch (Throwable $exception) {
        $span->recordException($exception);
        $span->setStatus('error');
        throw $exception;
    } finally {
        $span->end();
    }
}
~~~

The finally block is the important part. A timeout, exception, early return, or client disconnect must not leave a span open indefinitely. The adapter supplies the current-context behavior and exporter integration. The application supplies a low-cardinality operation name and safe attributes.

## Practical Example

An application operation can create a parent span and nest boundary spans:

~~~php
<?php

declare(strict_types=1);

function chargeCheckout(
    Tracer $tracer,
    BillingClient $billing,
    string $orderId,
): string {
    return inSpan($tracer, 'checkout.charge', function (Span $span) use (
        $tracer,
        $billing,
        $orderId,
    ): string {
        $span->setAttribute('app.operation', 'checkout');

        return inSpan(
            $tracer,
            'billing.charge',
            function (Span $billingSpan) use ($billing, $orderId): string {
                $billingSpan->setAttribute('billing.operation', 'charge');

                return $billing->charge($orderId);
            },
        );
    });
}
~~~

BillingClient is an application-level port in this example. A real adapter should create the child span as the current context before the client sends bytes, inject the current context into the request, record a bounded outcome, and end the span when the documented response boundary is reached. The operation ID or order ID may belong in a protected log or durable record; do not put a raw customer or payment identifier on every span by default.

## Production Example: Request to Queue

Consider an HTTP request that accepts a report and enqueues work:

~~~text
server span: POST /reports
├── internal span: validate report request
├── database span: create report operation
└── producer span: report.generate
        ↓ propagation carrier
consumer span: report.generate
├── database span: read report inputs
├── internal span: render report
└── producer span: report.ready
~~~

The request trace may end when the queue accepts the message. The consumer can continue the trace when that represents one causal workflow, or start a new trace linked to the producer when retention, sampling, or workflow lifetime makes that clearer. Either way, persist an operation ID and expose it in safe logs and status responses. A trace backend is not the durable source of truth for whether the report exists.

For a batch consumer, do not create one enormous trace containing thousands of messages. Use a bounded batch span with links or a sample of representative message contexts, and retain per-message state in the queue and operation records. The diagnostic design should reflect the batch's memory, retention, and query limits.

## Retries and Partial Failure

Retries can appear in two valid ways:

~~~text
logical span: billing.charge
├── attempt span: HTTP attempt 1 → timeout
└── attempt span: HTTP attempt 2 → success
~~~

The logical span answers how the application operation behaved. Attempt spans answer which physical calls consumed time and capacity. If the instrumentation only exposes one client span per logical call, record a bounded resend count and retry outcome instead of pretending there was no retry. Do not create an unbounded event for every internal loop iteration.

A missing child span does not prove that no work happened. The library may not be instrumented, the sampler may have dropped it, or export may have failed. Use request metrics and logs to detect population-wide behavior, then investigate representative traces. If a required business effect is missing from a trace, consult the durable operation record.

If the parent request times out while a child provider continues, the trace may contain a late child or no completed child depending on the instrumentation boundary. Record timeout and cancellation semantics explicitly. A span ending is an observation boundary, not proof that a remote server stopped processing.

## Sampling

Tracing every span for every request can exceed application, network, storage, and query budgets. Sampling chooses which traces or spans are retained or exported. Common strategies include:

* head sampling, decided near the trace root before the full outcome is known;
* parent-based sampling, which keeps child decisions consistent with the parent;
* tail sampling, which decides after a collector has observed enough of a trace;
* policy-based retention for errors, slow traces, rare operations, or selected deployments.

Sampling changes the meaning of a trace search. “No trace found” can mean the request was not sampled, not that it did not occur. Keep unsampled request counts, error ratios, and latency histograms in metrics. Preserve a safe operation or correlation ID in logs when the incident workflow requires joining evidence.

Sampling rules must be bounded and reviewable. Do not sample all traffic for a tenant by copying an unbounded tenant identifier into a backend rule without considering privacy and cost. A temporary diagnostic policy should have an owner, expiry, and rollback path.

## Span Attributes and Events

Attributes describe bounded facts known about the span:

~~~text
http.request.method = POST
http.route = /checkout
server.address = billing.internal
error.type = timeout
~~~

Events describe something that occurred during the span, such as a retry classification or a validation failure. Neither attributes nor events should contain passwords, cookies, access tokens, full request bodies, raw SQL with secrets, or uncontrolled user content. Use bounded error categories and normalized operation names.

Follow the semantic conventions of the chosen telemetry system for HTTP, database, messaging, and deployment attributes. A convention may define units, status behavior, span names, and required fields. If an application adds custom attributes, prefix or namespace them according to the convention and document their allowed values.

Do not set span status to error for every expected business rejection. A declined card or invalid form can be a successful execution of a business rule, even though it is a non-success response. Classify it with a bounded outcome attribute; reserve error status for failures according to the operation's contract.

## Correlation with Logs and Metrics

Use the trace ID and current span ID as safe correlation metadata in structured logs. Chapter 260 described request and operation identity; Chapter 261 described metrics. Correlation lets an operator move from an aggregate metric to a representative trace and then to bounded logs.

Do not put trace ID, span ID, or operation ID into metric labels by default. Those values are designed to be unique and would explode time-series cardinality. A metrics backend may support exemplars or trace references without turning every trace into a new series.

Align the boundaries:

| Signal | Best question |
| --- | --- |
| Metric | How widespread, frequent, or large is the condition? |
| Trace | Which path and dependency contributed to this operation? |
| Log | What bounded decision or failure evidence was recorded? |
| Durable record | What business state or workflow outcome was committed? |

If the signals disagree, investigate collection boundaries before inventing a system failure. A trace may end at response headers while the log is written during cleanup; a metric may count attempts while the durable record counts logical effects.

## Performance and Capacity

Every span can allocate objects, copy attributes, capture timestamps, enqueue export data, and consume backend storage. Instrumentation overhead is part of the request budget. Measure it under representative concurrency and payload sizes.

Bound the number of active spans and queued export records. A large trace can consume a PHP worker's memory, especially when a long-running job accumulates events or a batch contains too many child spans. End spans promptly, avoid retaining request bodies, and flush with a bounded policy during worker shutdown.

Sampling reduces retained volume but does not make instrumentation free. A head sampler may still create enough local state to make its decision; a tail sampler moves buffering into the collector. Choose where the memory and failure boundary belongs. Export asynchronously only when the SDK and operational contract make loss acceptable.

Trace timestamps from different hosts are not a perfect total order. Clock skew can make a child appear to begin before its parent. Use parent relationships and durations for causal structure, and use synchronized clocks plus server timing metadata to improve cross-host interpretation. Do not infer business ordering from visual position alone.

## Security and Privacy

Trace backends often contain request paths, hostnames, error categories, database metadata, and message attributes. Treat them as sensitive operational data. Restrict query access, encrypt transport and storage, define retention, and audit administrative access.

Never trust inbound trace context as identity. An attacker can submit a chosen trace ID to make their request harder to distinguish or to collide with another diagnostic search. Validate format, replace malformed values, and keep authorization independent. Avoid returning internal trace data to an unauthorized caller.

Redact before export where possible. A backend-side redaction rule is defense in depth, not permission to capture secrets in the application. Test synthetic tokens, cookies, authorization headers, raw query strings, uploaded content, and exception messages for absence from exported spans.

## Database and Messaging Boundaries

Database spans should identify a bounded logical operation, database role, and outcome. Avoid raw SQL text and literal values unless a protected diagnostic path explicitly permits them. Query fingerprints can help group behavior, but their normalization and privacy properties must be tested.

For messages, distinguish publish, delivery, processing, acknowledgment, retry, and dead-letter spans. A successful publish does not prove a consumer applied the business effect. A consumer span ending before acknowledgment can be redelivered. Use the queue's durable state and idempotency record to resolve these cases; use the trace to show where time and failure occurred.

Trace context in a message must not be the only message identity. Keep a durable message ID and operation ID with the message contract. A trace can be sampled or retained for a shorter period than the message's retry and reconciliation window.

## Concurrency and Workers

One trace can contain concurrent child operations. The visualization may show overlapping spans, but overlap does not prove that the underlying resources were independent. Correlate with connection-pool, worker, and dependency metrics to understand contention.

Do not store a mutable current span in a process-global singleton without a context mechanism that supports the runtime. In a synchronous FPM request, a request-scoped context is usually sufficient. In a long-running worker or Fiber-based design, context must follow the execution unit and be restored after suspension or message completion. Always close spans on success, exception, timeout, cancellation, and forced shutdown where cleanup can run.

## Testing

Test tracing as a contract, not as an assertion that every line creates a span:

* an inbound request creates one server boundary with a bounded name;
* an outbound call creates a child or linked span with the expected kind;
* valid context is propagated and malformed context is replaced safely;
* missing context creates a new root;
* retries distinguish logical work from physical attempts;
* exceptions record bounded error evidence and still end the span;
* queue consumers clear context between messages;
* forbidden values are absent from attributes, events, and exporter payloads;
* sampling does not change business counters or durable outcomes;
* exporter delay or failure cannot block the request without a documented policy.

Use an in-memory fake tracer to assert names, relationships, lifecycle, and redaction. Use integration tests to verify HTTP header injection, queue metadata, collector export, sampling behavior, and multi-process aggregation. Test a representative PHP-FPM request and a long-running worker separately; their lifetimes and cleanup risks differ.

## Common Mistakes

* Creating spans for every function and drowning useful boundaries in noise.
* Treating a trace ID as authentication or authorization.
* Putting user IDs, raw URLs, SQL, exception text, or request bodies on spans.
* Assuming “no trace found” means “no request happened.”
* Propagating context without propagating deadlines or cancellation semantics.
* Keeping a request's current span alive across unrelated queue messages.
* Counting retries as independent user operations.
* Assuming a completed producer span proves consumer work completed.
* Exporting synchronously without a timeout or drop policy.
* Using trace or span IDs as metric labels.
* Ignoring clock skew when interpreting cross-host timelines.
* Relying on backend redaction after secrets have already left the PHP process.

## Senior Engineer Thinking

The senior question is not “where can we add another span?” It is “which boundary would let an operator distinguish a local failure, a dependency delay, queue waiting, resource contention, and unknown remote completion?”

A useful trace is intentionally incomplete but semantically honest. It names the operation, shows the important boundaries, carries context safely, records bounded outcomes, and makes sampling visible. Metrics establish prevalence, traces explain representative paths, logs preserve bounded evidence, and durable records establish business truth.

When a trace is missing or contradictory, inspect instrumentation coverage, sampling, exporter health, process lifetime, clock behavior, and boundary definitions before changing application logic. Observability is itself a distributed system with capacity, failure, privacy, and recovery requirements.

## Exercises

1. Draw a trace for an HTTP checkout request that calls a database, retries a billing provider, and publishes an invoice event. Mark server, internal, client, producer, and consumer spans.
2. Design propagation metadata for an asynchronous report job. Decide which values belong in message metadata, durable operation state, logs, and traces.
3. Review a proposed span containing user_id, raw URL, SQL text, and exception message. Replace it with bounded attributes and identify where detailed diagnostics belong.
4. Build a fake tracer and test that a throwing handler records an error and ends its span, while a worker resets context before processing the next message.

## Review Questions

* What does a span represent, and why should spans follow meaningful boundaries?
* How do trace context propagation, authentication, and deadline propagation differ?
* When are span links more accurate than one parent span?
* Why must retries distinguish logical operations from physical attempts?
* What does “no trace found” mean when sampling is enabled?
* Why should trace IDs not be metric labels?
* How do PHP-FPM requests and long-running workers differ for context cleanup?
* Which evidence belongs in a durable operation record rather than a trace?
* How can tracing itself harm latency, memory, privacy, or availability?

## Summary

Tracing connects timed spans across PHP-FPM requests, application boundaries, databases, providers, queues, and workers. Use meaningful low-cardinality span names, validate and propagate context safely, distinguish parent relationships from links, separate logical operations from retry attempts, and make sampling visible. Correlate traces with metrics and structured logs without using unique IDs as metric labels. Treat exporters, buffers, clocks, worker cleanup, privacy, and retention as production constraints. A trace explains a representative path; metrics show prevalence, logs preserve bounded evidence, and durable records establish business truth.

## References

- [OpenTelemetry: Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [OpenTelemetry: Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [OpenTelemetry: Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- [OpenTelemetry PHP instrumentation](https://opentelemetry.io/docs/languages/php/instrumentation/)
- [OpenTelemetry PHP propagation](https://opentelemetry.io/docs/languages/php/propagation/)
- [OpenTelemetry HTTP span semantic conventions](https://opentelemetry.io/docs/specs/semconv/http/http-spans/)
- [Chapter 224 — Measuring Performance](../15-performance/224-measuring-performance.md)
- [Chapter 238 — Timeouts](../16-distributed-systems/238-timeouts.md)
- [Chapter 260 — Logging](./260-logging.md)
- [Chapter 261 — Metrics](./261-metrics.md)
- [Chapter 263 — Health Checks](./263-health-checks.md)
