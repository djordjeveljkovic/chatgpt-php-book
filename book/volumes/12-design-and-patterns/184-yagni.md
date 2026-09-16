---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 184
title: YAGNI
slug: yagni
status: complete
summary: ../../_ai/chapter-summaries/184-yagni-summary.md
---

# Chapter 184 — YAGNI

## Why This Matters

YAGNI means “You Aren't Gonna Need It.” Build the behavior required by a current, evidenced requirement instead of speculative features. Every unused option adds code, tests, documentation, configuration, attack surface, and decisions that future maintainers must understand.

YAGNI does not mean ignoring known requirements or refusing useful seams. Authentication, authorization, validation, observability, migrations, and a small interface around a volatile provider may be necessary before the first feature ships. The discipline is to separate evidence from imagination and to keep the cost of future change visible.

## Evidence Before Extension

Classify a proposed capability:

- required by an accepted use case or contract;
- required by a safety, legal, or operational control;
- likely based on measured usage or a committed roadmap;
- merely possible because a framework makes it easy.

Build the first two categories. Investigate the third with a small design or spike. Defer the last category until evidence changes. A backlog entry and a clear extension seam are often cheaper than a dormant implementation.

A feature flag for a real rollout or rollback plan is useful. A dozen flags for imagined tenants create combinations that no one tests and behavior that no one can explain.

## A Direct First Design

Suppose the current application sends one type of reservation email. A generic plugin registry with discovery, priorities, configuration schemas, and lifecycle hooks is speculative if no second email provider or template family exists.

~~~php
<?php

declare(strict_types=1);

interface Mailer
{
    public function send(string $recipient, string $template, array $data): void;
}

final class ReservationNotifier
{
    public function __construct(private Mailer $mailer)
    {
    }

    public function confirmed(string $email, string $reservationId): void
    {
        $this->mailer->send($email, 'reservation-confirmed', [
            'reservation_id' => $reservationId,
        ]);
    }
}
~~~

The Mailer boundary is justified because sending is an external effect that needs a production adapter and a test double. The class does not expose provider selection, template discovery, or future channels. When SMS or multiple providers becomes a requirement, add the smallest policy that expresses that choice.

## Necessary Seams versus Speculative Features

A seam controls a known boundary; a speculative feature implements behavior nobody needs yet. Injecting a clock into expiry logic can make a current requirement testable. Adding a ClockProviderFactory with time-zone strategies for every jurisdiction before the product supports them adds design cost without current value.

Design for change at stable boundaries:

- use an interface when a real external system or independent implementation exists;
- keep a command or service boundary where authorization and transactions belong;
- store a version when the data contract must evolve;
- expose an operation identifier when retries or reconciliation require it;
- keep private details private when no client depends on them.

Do not add an interface merely because a class might someday have another implementation. Concrete code can be refactored when evidence arrives, especially when tests protect its current behavior.

## Cost of Speculation

Speculative code has ongoing costs:

- more branches and combinations to test;
- configuration that can be invalid or insecure;
- larger deployment and support surfaces;
- unclear ownership and documentation;
- compatibility promises that become hard to remove;
- security review for paths that may never be used.

Estimate an option's cost over its lifetime, not only the hours to type it. A dormant admin endpoint can become an authorization vulnerability; an unused parser can become a supply-chain or resource risk. Removing unused code is often a security and operational improvement.

## YAGNI and Security

Security requirements are not speculative because an attacker does not wait for product demand. Validate input, authorize actions, protect secrets, set timeouts, limit resource consumption, and record the events needed to investigate. These controls should be designed with the current threat model even when the product feature is small.

Avoid “future-proof” cryptography or access controls that are not understood by the team. Use a maintained standard and document the current key, session, and credential lifecycle. A simple, reviewed policy is safer than a flexible policy with untested combinations.

## YAGNI and Data Design

Do not add ten nullable columns for imagined workflows. Model the current invariant and choose a migration path that can support a known next step. Preserve data that must be retained for legal, audit, or recovery reasons; deleting required history in the name of simplicity is not YAGNI.

For APIs, avoid exposing internal fields “in case clients need them.” Every public field can become a compatibility promise. Return the representation required by current clients and add fields deliberately under a documented evolution policy.

## Feedback Loops and Reversible Decisions

Keep decisions reversible when uncertainty is high:

1. Write the smallest vertical slice that proves the workflow.
2. Instrument adoption, latency, failures, and resource use.
3. Put an explicit owner and review date on experiments and flags.
4. Remove an unused path after evidence, not after wishful cleanup.
5. Add a durable abstraction only when multiple consumers or a real boundary justify it.

A reversible decision is not an excuse to skip data migration or security review. It means the design has a defined way to roll back, disable, or replace the behavior without corrupting state.

## Testing and Maintenance

Test current behavior and required failure paths. Do not create a large test matrix for options that do not exist. When a requirement is deferred, record what evidence would trigger implementation and keep the interface or data migration plan small.

Static analysis, dependency checks, and observability remain valuable even for a small feature because they protect the current system. YAGNI reduces speculative product behavior; it does not remove engineering hygiene.

## Common Mistakes

- Building plugin systems before a second plugin exists.
- Adding public API fields or options “for future clients.”
- Confusing a testable seam with an unneeded feature.
- Treating security controls as optional future work.
- Keeping permanent feature flags and dead branches.
- Designing schemas around imagined workflows.
- Skipping instrumentation because the first version is small.

## Senior Engineer Thinking

YAGNI is evidence-driven scope control. Build the current contract with the seams and controls its real boundaries require, keep uncertain decisions reversible, measure usage, and add flexibility when a concrete change justifies it. The goal is a system that can evolve, not a system that predicts every future feature.

## Exercises

1. Review a proposed plugin architecture and list the evidence required before implementing it.
2. Separate necessary security controls from speculative product options in a new API endpoint.
3. Design a feature flag lifecycle with owner, metrics, expiry date, and removal plan.
4. Choose a concrete seam for an external provider and explain why it is justified today.

## Review Questions

1. What is the difference between a necessary seam and speculative functionality?
2. Why do unused options create security and operational costs?
3. Which security controls remain necessary under YAGNI?
4. How can a team keep an uncertain decision reversible?
5. Why can public API fields be expensive to add early?

## Summary

YAGNI keeps implementation aligned with evidenced requirements while preserving the seams needed for real boundaries, security, and recovery. Build a small vertical slice, measure it, make uncertain decisions reversible, and add extension points when a concrete variation appears. Remove dormant complexity and treat every public option, data field, and feature flag as a maintenance and compatibility cost.

## References

- [Extreme Programming Explained: YAGNI](http://www.extremeprogramming.org/rules/youarentgonnaneedit.html)
- [Martin Fowler: You Aren't Gonna Need It](https://martinfowler.com/bliki/Yagni.html)
- [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/)
- [PHP interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)

