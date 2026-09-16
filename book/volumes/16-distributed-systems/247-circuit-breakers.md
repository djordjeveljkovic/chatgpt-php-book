---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 247
title: Circuit Breakers
slug: circuit-breakers
status: complete
summary: ../../_ai/chapter-summaries/247-circuit-breakers-summary.md
---

# Chapter 247 — Circuit Breakers

## Why This Matters

A circuit breaker stops sending work to a dependency that is failing or too slow. It prevents every caller from consuming a worker, connection, and timeout while waiting for an unhealthy service. It also gives the dependency a chance to recover.

A breaker is a protective control loop, not a health oracle. An open breaker means this caller's policy is refusing new attempts; it does not prove the dependency is globally down. Configure it from observed failures and the business contract, and provide a fallback or explicit error.

## States

The usual state machine is:

```text
             failure threshold
closed ─────────────────────→ open
  ↑                              │
  │ probe succeeds               │ cooldown
  │                              ↓
  └──────────── half-open ←──────┘
                 │
                 └─ probe fails → open
```

* **closed:** calls are allowed and outcomes are measured;
* **open:** calls fail fast or use a fallback;
* **half-open:** a small number of probes test recovery.

The transition policy needs a window, threshold, cooldown, and probe limit. Count timeout and connection failures differently from invalid requests or authorization failures. A bad caller request should not open a circuit for a healthy provider.

## Fail Fast and Fallback

When open, return a bounded dependency-unavailable result. Do not silently return empty authorization, payment success, or inventory availability. Optional recommendations may be omitted; required security decisions should fail closed.

Fallbacks have their own contract:

```text
provider unavailable → cached public description, marked stale
provider unavailable → pending workflow
provider unavailable → explicit error
```

Never let the fallback make an unauthorized decision. A stale cache must be scoped and have a maximum age.

## A Small Policy

The example models the state transition; synchronization and metrics belong to the deployment:

~~~php
<?php

declare(strict_types=1);

enum CircuitState: string
{
    case Closed = 'closed';
    case Open = 'open';
    case HalfOpen = 'half_open';
}

final class CircuitBreaker
{
    private CircuitState $state = CircuitState::Closed;
    private int $failures = 0;
    private int $openedAtMs = 0;
    private bool $probeInFlight = false;

    public function __construct(
        private readonly int $failureThreshold,
        private readonly int $cooldownMs,
    ) {
        if ($failureThreshold < 1 || $cooldownMs < 1) {
            throw new InvalidArgumentException('Invalid circuit policy');
        }
    }

    public function allow(int $nowMs): bool
    {
        if ($this->state === CircuitState::Closed) {
            return true;
        }

        if ($this->state === CircuitState::Open
            && $nowMs - $this->openedAtMs >= $this->cooldownMs
        ) {
            $this->state = CircuitState::HalfOpen;
        }

        if ($this->state !== CircuitState::HalfOpen || $this->probeInFlight) {
            return false;
        }

        $this->probeInFlight = true;
        return true;
    }

    public function recordSuccess(): void
    {
        $this->failures = 0;
        $this->state = CircuitState::Closed;
        $this->probeInFlight = false;
    }

    public function recordFailure(int $nowMs): void
    {
        $this->failures++;
        if ($this->failures >= $this->failureThreshold) {
            $this->state = CircuitState::Open;
            $this->openedAtMs = $nowMs;
            $this->probeInFlight = false;
        }
    }
}
~~~

The `allow()` method is not safe for multiple processes without an atomic shared state. A process-local breaker protects only one process. A shared breaker can coordinate more callers but adds a dependency and synchronization cost. Many systems use local breakers intentionally to stop each replica from repeatedly calling a failing dependency.

## Failure Windows

A count threshold reacts differently from a percentage threshold. Five failures in five calls is severe; five failures in ten thousand may not be. A time window prevents failures from remaining relevant forever. Also consider consecutive failures, timeout duration, and dependency-specific status categories.

Do not open a circuit on every isolated slow call when the dependency's normal latency is variable. Use a timeout aligned with the parent deadline and record the reason. A circuit that opens too aggressively causes unnecessary outages; one that opens too slowly does not protect capacity.

## Half-Open Probes

Allowing every waiting request through when the cooldown ends defeats the breaker. Permit one or a small bounded number of probes. The rest fail fast or use the fallback. A successful probe should not immediately unleash a large backlog; ramp up gradually.

Multiple replicas may enter half-open simultaneously. That is acceptable if their combined probe rate fits the provider's recovery capacity. A globally coordinated breaker reduces probes but can itself become a failure dependency.

## Circuit Breakers and Retries

Place the breaker around the operation whose failures it measures, and coordinate it with retry policy. A retry inside an open circuit should not bypass the circuit. Conversely, a breaker that counts each retry as an independent failure may open faster than intended.

```text
parent deadline
  → breaker admission
  → bounded retry attempts
  → dependency call
```

The adapter should classify errors consistently. Do not count caller validation failures as provider failures. See [Chapter 239 — Retries](./239-retries.md) and [Chapter 240 — Backoff](./240-backoff.md).

## Observability

Record state changes, rejected calls, probe outcomes, fallback outcomes, dependency latency, and the failure category. Include operation and release dimensions, but avoid raw user IDs and sensitive payloads. Alert on sustained open time and customer-visible fallback or error rates, not only on a state transition.

An open circuit can hide a provider recovery if no probes are allowed or if the health check does not represent the real operation. Keep the recovery policy observable and bounded.

## Testing

Inject a clock and a dependency outcome sequence. Test threshold boundaries, cooldown boundaries, successful and failed half-open probes, concurrent probe limits, fallback authorization, retry interaction, and reset after success. Test that invalid input and caller authorization failures do not open the provider circuit.

Run a failure-injection test with realistic concurrency and confirm that worker and connection occupancy fall when the circuit opens. Test a breaker store outage if state is shared and define whether the local policy fails open or closed.

## Security

A circuit breaker is not authentication or authorization. Protect administrative reset endpoints and do not let an ordinary user force a global circuit closed. Avoid returning internal dependency names or topology in public errors.

## Common Mistakes

* Treating an open circuit as proof that the dependency is globally down.
* Counting invalid requests as dependency failures.
* Letting every request through during half-open recovery.
* Allowing retries to bypass circuit admission.
* Returning an unsafe empty or affirmative fallback.
* Using a process-local state as a global invariant without analysis.
* Tuning thresholds without measuring normal latency and error shape.
* Alerting only when the circuit opens and not on fallback impact.

## Senior Engineer Thinking

Ask which resource the breaker protects, which outcomes count as dependency failure, and what the caller receives while the circuit is open. The breaker should reduce damage and make recovery possible; it should not conceal a required operation or create a new single point of failure.

## Exercises

1. Choose thresholds and fallback policies for recommendations, authorization, and payment capture.
2. Test a circuit with five failures, a cooldown, a failed probe, and a successful probe. Verify every state.
3. Calculate the maximum probe rate across ten application replicas.
4. Integrate a breaker with a retry budget and prove that retries cannot bypass open state.

## Review Questions

* Which states does a circuit breaker use, and why is half-open bounded?
* Which failures should not open a dependency circuit?
* Why can a fallback be a security risk?
* How should breakers and retries coordinate?
* What does a process-local breaker protect?
* Which metrics show customer impact while a circuit is open?

## Summary

A circuit breaker fails fast when a dependency's measured failures or latency threaten caller capacity. Use closed, open, and bounded half-open states; classify failures; coordinate with deadlines, retries, and fallbacks; protect required decisions from unsafe degradation; and observe state, probes, and user-visible outcomes. Treat the breaker as a local protective policy, not a perfect health oracle.

## References

- [Microsoft: Circuit Breaker pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Google SRE: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Chapter 239 — Retries](./239-retries.md)
- [Chapter 240 — Backoff](./240-backoff.md)
