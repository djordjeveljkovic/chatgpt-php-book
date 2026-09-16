---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 241
title: Partial Failure
slug: partial-failure
status: complete
summary: ../../_ai/chapter-summaries/241-partial-failure-summary.md
---

# Chapter 241 — Partial Failure

## Why This Matters

In a single process, a failure often stops the operation at one visible point. In a distributed system, one participant can fail while others continue. A database write may commit while the response is lost, one provider may time out while another succeeds, or a queue consumer may die after performing an effect but before acknowledging the message.

Partial failure is normal behavior at a network boundary. Reliable systems do not try to make it impossible; they make the possible states explicit, preserve enough evidence to recover, and avoid turning uncertainty into a second destructive action.

## Mental Model

Model an operation as a state machine rather than a boolean:

```text
created → running → succeeded
    ↘       ↓          ↑
     rejected  unknown ─┘
```

`unknown` means the observer cannot determine whether the owner completed the work. It is not the same as `failed`. The owner may expose an operation lookup, an event, or a reconciliation process that converts unknown into a known state.

For a fan-out request:

```text
request
  ├─ inventory: success
  ├─ pricing: timeout
  └─ recommendations: success
```

The application needs a policy for required and optional branches. Returning a complete-looking response with missing authorization or inventory data is worse than returning a visible partial result.

## Failure Categories

Classify failures by what the system knows and what can be done:

* **rejected before execution:** validation or authorization failed;
* **unavailable:** the dependency could not accept or complete work;
* **timed out:** the observer stopped waiting;
* **partially completed:** some branches or steps succeeded;
* **unknown completion:** an effect may have occurred;
* **corrupt or incompatible:** the message or response cannot be safely interpreted.

The category should drive the next action. A rejected request should not be retried. An unavailable read may use a bounded fallback. An unknown payment needs lookup or reconciliation. A corrupt message needs quarantine and investigation.

## Ownership and Evidence

The service that owns an invariant must be able to decide its state. Callers should not infer “paid” from receiving an HTTP 200 from an intermediary, or infer “unpaid” from a client timeout. Persist an operation ID, state, timestamps, relevant version, and safe failure category where the workflow needs recovery.

Evidence must have a retention and privacy policy. Store enough to reconcile without retaining credentials or complete sensitive payloads forever. An operation lookup should authenticate the caller and enforce tenant ownership; an ID alone is not authorization.

## Partial Results

Partial results are a product contract. They are appropriate when the missing data is optional and the response says what is missing. They are unsafe when omission changes the meaning of a security, financial, or inventory decision.

```text
required branch fails → explicit error or pending state
optional branch fails  → response with a visible omission
stale cache used       → bounded-staleness marker when relevant
```

Do not silently replace a failed authorization service with “allow.” Fail closed for permission decisions unless a separately designed emergency policy applies.

## A Result Type

Make known and unknown states difficult to confuse:

~~~php
<?php

declare(strict_types=1);

enum OperationState: string
{
    case Succeeded = 'succeeded';
    case Rejected = 'rejected';
    case Pending = 'pending';
    case Unknown = 'unknown';
}

final readonly class OperationResult
{
    public function __construct(
        public string $operationId,
        public OperationState $state,
        public ?string $reason = null,
    ) {
        if ($operationId === '') {
            throw new InvalidArgumentException('Missing operation identity');
        }
    }
}

function publicStatus(OperationResult $result): int
{
    return match ($result->state) {
        OperationState::Succeeded => 200,
        OperationState::Rejected => 422,
        OperationState::Pending => 202,
        OperationState::Unknown => throw new LogicException(
            'Unknown completion requires an explicit API contract',
        ),
    };
}
~~~

The status mapping is illustrative. The important property is that `Unknown` is represented explicitly and is not converted to a successful or failed business result by accident.

## Multi-Step Workflows

A workflow that updates several authorities cannot assume one local transaction covers them all:

```text
reserve inventory → authorize payment → create shipment
       success              timeout
```

Persist each step and its operation identity. Define compensation or reconciliation for every transition. Releasing inventory may compensate a payment authorization, but compensation itself can fail and need retry. A workflow coordinator can own progress without owning every domain invariant; each service must still validate its own command.

Avoid presenting a multi-step operation as atomically complete when it is only eventually consistent. Use `pending`, a status endpoint, or a durable event. The client can show a useful state while the system finishes.

## Failover and Recovery

Recovery changes the observer's knowledge over time. A stale replica may show the old state; a delayed event may arrive after a timeout; a restarted worker may replay a message. State transitions need version or identity checks so late information cannot overwrite a newer decision.

Reconciliation should be safe to run repeatedly. Read the owner of the fact, compare operation IDs and versions, and repair derived state. Do not “repair” by issuing an unbounded new side effect.

## What PHP Does

PHP exceptions represent local observations. A `TransportException` cannot tell the application whether a remote server read the request unless the protocol provides that evidence. In PHP-FPM, the worker remains occupied while a blocking call waits; in a queue worker, a crash can cause redelivery.

Use `finally` to release local resources, but do not confuse local cleanup with remote cancellation. Propagate deadlines and reset request context in long-running processes. See [Chapter 238 — Timeouts](./238-timeouts.md) and [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md).

## Testing

Test each boundary state, not only success and exception:

* response lost after a committed write;
* one branch succeeds and another times out;
* a compensation fails and is retried;
* a stale event arrives after a newer state;
* a worker dies before acknowledgment;
* a dependency returns malformed or incompatible data;
* the reconciliation job runs twice.

Use fakes for deterministic state transitions and contract tests for the real message format. A failure-injection test should assert business outcomes, durable records, duplicate behavior, and operator-visible evidence.

## Security

Partial failure must not create a fail-open path. Protect operation lookups with authentication and object-level authorization. Avoid exposing whether another tenant's operation exists. Redact sensitive data from reconciliation logs and dead-letter records.

## Common Mistakes

* Treating a timeout as proof that an effect did not happen.
* Returning a complete response after a required security or inventory branch failed.
* Making a multi-service workflow look transactional when it is not.
* Letting a stale event overwrite a newer state.
* Retrying compensation without a bound or identity.
* Storing no evidence for an operation that needs reconciliation.
* Using a status endpoint that accepts any operation ID without tenant checks.

## Senior Engineer Thinking

Ask: “what does each participant know, what has definitely happened, and what remains uncertain?” Then design a state machine that makes uncertainty visible and a recovery action that is safe when run repeatedly. Distributed reliability comes from explicit knowledge and convergence, not from pretending the network is a transaction manager.

## Exercises

1. Model a checkout with inventory, payment, and shipment as a state machine. Mark every ambiguous transition and its reconciliation action.
2. Design a partial response for a product page with optional recommendations and required authorization.
3. Inject a lost response after a database commit and verify that a second client request does not duplicate the effect.
4. Define the evidence and authorization rules for an operation-status endpoint.

## Review Questions

* Why is unknown completion different from failure?
* Which service owns the truth for an invariant?
* When is a partial response safe?
* Why can compensation itself need retries and identity?
* How can a stale event damage a recovered workflow?
* Which state must reconciliation preserve?

## Summary

Partial failure is expected when independent processes and networks participate in one workflow. Represent pending, rejected, succeeded, and unknown states explicitly; preserve operation identity and evidence; make partial results a deliberate contract; reconcile from the owning service; and protect every recovery action with idempotency, authorization, deadlines, and bounded retries.

## References

- [Google SRE: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [AWS Builders' Library: Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [Chapter 238 — Timeouts](./238-timeouts.md)
- [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md)
