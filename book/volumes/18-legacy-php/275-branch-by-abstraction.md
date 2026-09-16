---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 275
title: Branch by Abstraction
slug: branch-by-abstraction
status: complete
summary: ../../_ai/chapter-summaries/275-branch-by-abstraction-summary.md
---

# Chapter 275 — Branch by Abstraction

## Why This Matters

When old and new implementations must coexist, changing every caller at once is often the riskiest part of the migration. Branch by Abstraction introduces one stable interface, places both implementations behind it, and changes the selection policy without forcing callers to know which path is active.

The abstraction is useful only if it represents a real capability contract. An interface that exposes legacy globals, array indexes, SQL quirks, and provider-specific errors merely relocates the coupling. The branch should reduce knowledge, preserve authority, and make differences observable.

## Mental Model

The caller depends on a port; the port selects one implementation:

~~~text
caller
  ↓ stable capability interface
selector ──→ legacy implementation
     └────→ new implementation
                    ↓
            shared or transitioned boundary
~~~

The branch has four independent questions:

* Which implementation supplies the user-visible answer?
* Which implementation may mutate state?
* Which implementation’s effects are authoritative?
* What evidence decides whether the branch can change or be removed?

Do not answer all four with one feature flag by accident. A new reader can be compared while the legacy writer remains authoritative. A new command can own writes while a compatibility reader serves old data. The plan must state the authority for each operation.

## Define the Abstraction

Start from the capability a caller needs, not the shape of the old implementation. A useful interface describes stable inputs, outputs, failure classification, and effect identity:

~~~text
poor abstraction: executeLegacyQuery(array $oldOptions): array
better abstraction: findInvoice(InvoiceId $id): InvoiceView
                  markPaid(InvoiceId $id, OperationId $op): PaymentResult
~~~

The better boundary hides query construction, table names, connection state, and legacy result arrays. It does not hide business decisions that callers must own, such as whether a missing invoice is a 404 or a retryable state.

Document:

* accepted input and version rules;
* output shape and missing-value semantics;
* errors, warnings, and retry classification;
* transaction and locking expectations;
* external-effect and idempotency rules;
* consistency and ordering requirements;
* observability fields and owner.

If the old implementation cannot satisfy the interface without changing behavior, create a compatibility adapter or narrow the interface. Do not silently coerce an incompatible result.

## Introduce the Abstraction Safely

Use a sequence that leaves callers working:

1. characterize the existing caller and implementation;
2. define the smallest stable interface;
3. wrap the old implementation without changing authority;
4. route all callers through the old adapter;
5. add the new implementation behind the same interface;
6. select the new path for controlled cases;
7. compare or reconcile results and effects;
8. remove the old adapter only after its retirement evidence exists.

The interface commit and implementation-switch commit should be reviewable separately. A branch is easier to diagnose when a failure identifies whether the contract, adapter, implementation, or selector changed.

## A Stable Read Abstraction

This example keeps callers independent of the legacy query shape:

~~~php
<?php

declare(strict_types=1);

final readonly class InvoiceView
{
    public function __construct(
        public string $id,
        public string $status,
        public int $totalCents,
    ) {
    }
}

interface InvoiceReader
{
    public function find(string $invoiceId): ?InvoiceView;
}

final class LegacyInvoiceReader implements InvoiceReader
{
    public function find(string $invoiceId): ?InvoiceView
    {
        global $db;

        $row = legacy_invoice_lookup($db, $invoiceId);
        if (!is_array($row)) {
            return null;
        }

        return new InvoiceView(
            id: (string) ($row['invoice_id'] ?? ''),
            status: (string) ($row['state'] ?? 'unknown'),
            totalCents: (int) ($row['amount_cents'] ?? 0),
        );
    }
}

final class NewInvoiceReader implements InvoiceReader
{
    public function find(string $invoiceId): ?InvoiceView
    {
        // The new adapter can use a typed repository or a new schema.
        return null;
    }
}

enum ReadPath: string
{
    case Legacy = 'legacy';
    case New = 'new';
}

final readonly class ReaderSelector
{
    public function __construct(
        private InvoiceReader $legacy,
        private InvoiceReader $new,
        private ReadPath $path,
    ) {
    }

    public function find(string $invoiceId): ?InvoiceView
    {
        return ($this->path === ReadPath::New ? $this->new : $this->legacy)
            ->find($invoiceId);
    }
}
~~~

The example is intentionally incomplete as an application: `NewInvoiceReader` is a placeholder adapter, and the cast rules require characterization before they are trusted. The interface is the migration seam. It does not prove that the new implementation has the same authorization, freshness, ordering, or error behavior.

## Selection Policy

Keep selection outside the capability implementation and make it deterministic:

~~~text
authorized request
       ↓
selection context: capability, tenant, environment, policy version
       ↓
legacy | new | shadow
       ↓
bounded decision evidence
~~~

The policy should have a safe default, an owner, a version, and an expiry or removal condition. Prefer explicit cohorts and stable keys over an unrepeatable random choice. Never let a client-controlled flag select a privileged implementation.

Selection should not automatically retry through the other implementation after an ambiguous write. A timeout can mean that the first implementation committed. Fallback is a new attempt, not an undo operation.

## Shadow Comparison

Shadowing is appropriate when the secondary implementation can run without forbidden effects:

~~~text
primary → user-visible result
   │
   └─ secondary → normalized result → comparison record
~~~

Compare semantic fields, not only serialized output. Include:

* authorization and tenant scope;
* missing, null, and default values;
* ordering, precision, and pagination;
* freshness and version information;
* error classification and warnings;
* query count and resource behavior where relevant.

The secondary path must not send mail, publish a message, charge a provider, mutate shared state, or alter cache entries unless those effects are explicitly isolated and part of the test. Record comparison volume and sampling so “no differences” is not mistaken for “the shadow ran.”

## Branching Writes

For writes, branch on authority rather than merely implementation:

| Mode | Read result | Writer | Secondary behavior |
| --- | --- | --- | --- |
| legacy | legacy | legacy | none |
| compare-read | legacy | legacy | new read-only |
| new-canary | new or defined | new for selected scope | legacy disabled for operation |
| new-authority | new | new | compatibility read only |
| retired | new | new | old adapter removed |

One operation should have one authoritative writer in a phase. If both systems must receive state, design an explicit synchronization mechanism with an operation ID, conflict policy, delivery guarantee, reconciliation process, and recovery owner. Calling both implementations from a wrapper does not provide those properties.

## Drift Classification

Differences need categories:

* representation drift: harmless only when the consumer contract permits it;
* data drift: missing, extra, stale, or differently interpreted state;
* policy drift: authorization, pricing, or business-rule difference;
* operational drift: latency, memory, query count, queue age, or effect count;
* environment drift: runtime, extension, configuration, timezone, or locale;
* harness drift: fixture, normalization, or comparison defect.

Do not set the branch to “new” because the aggregate mismatch rate is low. A single authorization or duplicate-payment difference can be a stop condition. Define severity by consequence and capability, not just percentage.

## Transactions and Effects

The abstraction must preserve transaction ownership. A repository adapter should not begin or commit a transaction owned by an application operation unless that is its explicit contract. A new implementation that writes an outbox record at a different point changes delivery behavior even if the main row matches.

For an effect, record attempted, accepted, confirmed, and unknown states separately. Give the operation a stable identity. If a branch fails after commit, route rollback cannot make the effect disappear. Use reconciliation or forward recovery, and keep the old implementation able to interpret compatible state until the recovery window closes.

## Runtime and Test Boundaries

Modern PHP interfaces and selectors can be useful around PHP 5 source, but their syntax cannot be loaded by PHP 5. Test the old adapter and the new implementation under the runtimes they claim to support. Compare the same fixture and configuration, but keep databases, files, sessions, caches, and providers isolated.

Static analysis can verify that both implementations satisfy the interface under the analyzed PHP version. It cannot prove dynamic include behavior, extension semantics, SQL modes, old warnings, or external completion. Chapter 272’s characterization evidence remains the baseline.

## Rollout and Retirement

A branch should have an explicit lifecycle:

~~~text
old adapter only
      ↓
new adapter installed
      ↓
shadow or controlled cohort
      ↓
new authority
      ↓
old calls and messages drained
      ↓
old adapter and selector branch removed
~~~

At each transition, update metrics, runbooks, alerts, deployment artifacts, and rollback decisions. Remove a branch only after proving that no supported caller, job, consumer, repair script, or recovery process depends on it.

## Common Mistakes

* Designing an interface from legacy implementation details.
* Switching all callers and implementations in one commit.
* Treating a feature flag as a complete authority or rollback policy.
* Running shadow code with real external effects.
* Fallback-retrying ambiguous writes through the other implementation.
* Comparing only JSON text while ignoring authorization, ordering, or precision.
* Allowing old and new implementations to write the same invariant independently.
* Treating low aggregate drift as proof that every case is safe.
* Testing modern adapters only under PHP 8 and claiming PHP 5 compatibility.
* Leaving selection branches, flags, and adapters without expiry or ownership.

## Senior Engineer Thinking

The senior question is not “how do we put an interface in front of both versions?” It is “what stable capability contract can both versions satisfy, which path owns each effect, and what evidence lets us change or remove the branch?”

Branch by Abstraction is valuable when the abstraction reduces caller knowledge and makes authority, drift, and recovery explicit. It is harmful when it disguises incompatible semantics behind a common name. Keep the branch narrow, the policy observable, and retirement part of the design from the beginning.

## Exercises

1. Design a stable interface for a legacy report reader without exposing SQL, globals, or array indexes. List its missing-value and error contracts.
2. Create a selection policy with old/new/shadow modes, a safe default, a stable cohort key, policy version, and stop conditions.
3. Build a drift-classification table for representation, data, policy, operational, environment, and harness differences.
4. Draw a transaction/effect timeline for a new writer and explain whether routing rollback can recover each effect.
5. Write retirement evidence for an abstraction branch, including HTTP, CLI, cron, queue, repair, and recovery callers.

## Review Questions

* What makes an abstraction stable during migration?
* Why should an interface describe a capability rather than a legacy query?
* Which authority questions must be separated for reads and writes?
* When is shadowing safe, and which effects must it prohibit?
* Why is fallback after an ambiguous write dangerous?
* How should behavioral drift be classified?
* What can an interface check that static analysis cannot prove?
* Why must transaction ownership remain explicit behind the abstraction?
* What evidence is needed before removing the old implementation?
* When does Branch by Abstraction add complexity without reducing coupling?

## Summary

Branch by Abstraction places old and new implementations behind a stable capability contract so callers can remain unchanged while selection evolves. Define inputs, outputs, errors, transactions, effects, and observability independently of legacy details; install the old adapter first; select deterministically; shadow only side-effect-free paths; keep one authoritative writer; classify drift by consequence; test runtime boundaries; and retire the branch after evidence. An interface is a migration seam only when it reduces knowledge and makes authority and recovery explicit.

## References

- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](./271-legacy-architecture.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 273 — Safe Refactoring](./273-safe-refactoring.md)
- [Chapter 274 — Strangler Pattern](./274-strangler-pattern.md)
- [Chapter 277 — Database Migration](./277-database-migration.md)
- [Chapter 278 — PHP Version Migration](./278-php-version-migration.md)
