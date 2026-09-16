---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 295
title: Technical Decision Making
slug: technical-decision-making
status: complete
summary: ../../_ai/chapter-summaries/295-technical-decision-making-summary.md
---

# Chapter 295 — Technical Decision Making

## Why This Matters

Senior technical decisions are explicit choices made under uncertainty, supported by proportionate evidence, bounded by risk, assigned to accountable owners, and revisited when assumptions change. Code review, architecture, debt, incidents, performance, and security all leave decisions with consequences for users, data, operations, and future change.

Waiting for perfect information is itself a decision. So is accepting a workaround, delaying a migration, keeping a modular monolith, or shipping a reversible experiment. Make the choice visible before implementation hardens it.

## Frame the Decision

Separate a decision from a task and from an unresolved question. Record the decision, desired outcome, context, constraints, non-goals, deadline, affected users and systems, options, reversibility, evidence, owner, and review condition.

“Improve search” is a goal. “Use a tenant-scoped database query for the next release, with a projection prototype before the largest tenant exceeds a stated size” is a decision with scope, sequence, and a reconsideration trigger.

## Facts, Constraints, and Unknowns

Label every important claim:

| Type | Meaning | Example |
| --- | --- | --- |
| fact | observed or documented | p99 is 1.8 seconds for large tenants |
| constraint | must be respected | old workers run during deployment |
| assumption | believed but unverified | the new index fits write capacity |
| hypothesis | testable explanation | lock waits cause tail latency |
| unknown | material missing information | provider completion after timeout |

Do not promote an assumption to a fact because it appears in a design document. State what observation would change the decision and who will gather it.

## Generate Real Options

Compare at least three credible choices, including inaction. For a PHP search system, options might be keeping and containing the database query, improving its query and indexes, adding a rebuildable cache or projection, or splitting the capability into a service. For notifications, compare synchronous delivery, an outbox and queue, and bounded deferral. For legacy code, compare containment, branch by abstraction, replacement, and retirement.

If only one option is written down, the team is usually defending a preference rather than making a decision.

## Evaluate Options Explicitly

Use a matrix and state the assumptions behind each assessment:

| Criterion | Query | Projection | Service split |
| --- | --- | --- | --- |
| correctness and isolation | direct authority | lag and rebuild controls | duplicated boundary risk |
| latency and capacity | database-bound | read-optimized | network and fan-out cost |
| recovery | simple source | rebuild and reconcile | multiple deployment domains |
| migration cost | low | medium | high |
| ownership | existing team | projection owner needed | new operational owner |
| reversibility | high | medium | low after contracts form |

Also evaluate security, availability, operability, future change, product impact, and financial cost. Numerical scores are aids to judgment, not objective truth.

## Gather Proportionate Evidence

Match evidence to uncertainty: characterization tests for legacy behavior, prototypes for integration risk, query plans and representative workloads for database assumptions, load tests for capacity, threat models and permission matrices for security, migration rehearsals for compatibility, and incident data for operational claims.

Ask, “What observation would make us choose another option?” Evidence should be the smallest useful experiment, not a demand for exhaustive certainty.

## Reversibility and Risk

Prefer reversible experiments and staged commitments. Feature flags, adapters, shadow reads, canaries, and bounded cohorts preserve options. Public APIs, schema ownership, deleted data, external contracts, credential formats, and service splits are harder to reverse and need stronger evidence, explicit approval, and recovery planning.

Risk includes likelihood, impact, exposure, uncertainty, detection quality, recovery cost, and blast radius. Include the cost of inaction: delaying a PHP upgrade may increase unsupported-package exposure, while rushing it may create a larger migration failure.

Do not reduce every decision to one number. Use numbers to expose assumptions, then apply judgment and accountable ownership.

## Architecture Decision Records

An ADR should preserve title, status, date, owner, context, decision, rejected alternatives, constraints, evidence and assumptions, quality attributes, risks and consequences, migration, rollback or forward recovery, observability, accepted debt, and review condition. It records why a boundary, data owner, effect model, or runtime choice was selected—not merely which classes were added.

Rejected alternatives matter because they prevent the next team from repeating the same debate without knowing what changed. Accepted risk needs an owner, signal, trigger, and review date.

## Decision Rights and Communication

Identify who provides input, who decides, who executes, and who accepts residual risk. Domain, security, database, operations, product, support, compliance, and customer stakeholders may have different authority. “Everyone agrees” is not accountability.

Engineers need constraints and implementation consequences; operators need rollout, alerts, and recovery; product leaders need impact, cost, timing, and alternatives; customers need behavior, limitations, and commitments. Communicate uncertainty without hiding or overstating it.

## Decision Traps

Watch for authority bias, anchoring, sunk-cost reasoning, false urgency, premature certainty, solution-first thinking, popularity mistaken for evidence, local-metric optimization, ignoring inaction, assuming rollback is safe, mistaking an experiment for a final commitment, and recording a choice without its assumptions.

Counter them with alternatives, disconfirming evidence, independent review for high-risk choices, and a review condition. A senior engineer can change a recommendation when evidence changes.

## PHP Case Study

Chapter 287’s tenant-scoped search has rising p99 latency for large tenants. A query is correct but database-bound; a cache reduces repeated work but requires complete tenant/version keys; a projection improves reads but adds lag, rebuild, queue, and ownership costs; a service split adds network, deployment, security, and operational boundaries.

The decision record compares isolation, stable ordering, transaction and projection semantics, PHP-FPM and worker capacity, queue behavior, mixed-version rollout, rollback/rebuild, ownership, and cost. The first decision may improve the query and add a projection seam, with a measurable trigger for a later split. The best answer is the one whose consequences the team can observe and operate.

## Workflow

1. Frame outcome, deadline, and affected scope.
2. Separate facts, constraints, assumptions, hypotheses, and unknowns.
3. Identify stakeholders and decision rights.
4. Generate credible options, including inaction.
5. Identify risk, irreversibility, and recovery boundaries.
6. Gather the smallest useful evidence.
7. Decide and record rationale, consequences, and accepted risk.
8. Communicate owners and next actions.
9. Roll out with guardrails and stop conditions.
10. Revisit when evidence, workload, dependencies, or priorities change.

## Common Mistakes

* treating a task or preference as a decision;
* writing one favored option and calling it analysis;
* confusing assumptions with facts;
* waiting for perfect information while risk compounds;
* using a score without stating its assumptions;
* ignoring the cost of inaction;
* treating a Git revert as recovery for data and external effects;
* choosing a service split before establishing ownership;
* accepting security, data, or operational risk without a named owner;
* recording what was chosen but not what was rejected or why;
* communicating certainty that the evidence does not support;
* never revisiting a decision after conditions change.

## Senior Engineer Thinking

Good technical decision making makes uncertainty visible, compares real options, gathers proportionate evidence, protects against irreversible mistakes, records ownership and consequences, and revisits decisions when reality changes. Confidence is not certainty; accountability is what makes a decision usable by the next team.

## Exercises

1. Write an ADR for a PHP runtime upgrade with mixed FPM and worker versions.
2. Compare synchronous notifications with an outbox and queue.
3. Decide whether to repay or contain a Chapter 290 debt item.
4. Build a decision matrix for the Chapter 287 search architecture.
5. Identify assumptions in an incident-recovery plan and design tests for them.
6. Design a reversible performance experiment with guardrails.
7. Assign decision rights among engineering, product, security, and operations.
8. Rewrite an overconfident recommendation using explicit uncertainty and a review condition.

## Review Questions

* What makes a decision different from a task or question?
* How should facts, constraints, assumptions, hypotheses, and unknowns be labeled?
* Why must the cost of inaction be an option?
* How do reversibility and sequencing affect required evidence?
* Which qualities belong in a technical decision matrix?
* Who accepts residual risk?
* Why should rejected alternatives and assumptions be recorded?
* Which cognitive traps distort technical judgment?
* What makes an experiment safer than a final commitment?
* When should an ADR be revisited?

## Summary

Technical decisions are explicit choices under uncertainty. Frame the outcome and constraints, separate facts from assumptions, compare real options and inaction, gather proportionate evidence, evaluate risk and reversibility, record rationale and rejected alternatives, assign decision rights, communicate consequences, roll out with guardrails, and revisit when reality changes.

## References

- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 272 — Characterization Tests](../../volumes/18-legacy-php/272-characterization-tests.md)
- [Chapter 280 — Tennis Reservation Service](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 289 — Architecture Review](289-architecture-review.md)
- [Chapter 290 — Technical Debt](290-technical-debt.md)
- [Chapter 291 — Debugging Production](291-debugging-production.md)
- [Chapter 292 — Performance Investigation](292-performance-investigation.md)
- [Chapter 293 — Security Review](293-security-review.md)
- [Chapter 294 — Incident Investigation](294-incident-investigation.md)

## Chapter 296 Handoff

No option is universally best. Chapter 296 will examine engineering trade-offs among correctness, simplicity, latency, availability, cost, consistency, security, and maintainability when every choice has consequences.
