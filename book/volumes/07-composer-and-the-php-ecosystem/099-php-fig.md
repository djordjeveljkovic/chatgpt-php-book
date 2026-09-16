---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 99
title: PHP-FIG
slug: php-fig
status: complete
summary: ../../_ai/chapter-summaries/099-php-fig-summary.md
---

# Chapter 99 — PHP-FIG

## Why This Matters

PHP frameworks, libraries, and applications are built by independent teams. They often need to exchange objects or cooperate across package boundaries, but they do not share a release schedule or internal design. If each project invents its own logging, HTTP-message, or autoloading contract, users must write adapters at every boundary. Shared conventions can reduce that coordination cost.

The PHP Framework Interoperability Group (PHP-FIG) provides a place for established PHP projects and contributors to discuss those common problems and publish recommendations. Its work matters because package authors can agree on a boundary without agreeing on a framework, runtime, or application architecture. The group does not make PHP itself enforce those agreements. A PSR is useful when projects choose to implement it and when the standard defines the behavior that matters to both sides.

This chapter explains the organization and how its recommendations are developed. Chapter 98 covered the exact PSR-4 mapping contract. [Chapter 100](100-psr-standards.md) catalogs the standards and their subject areas; this chapter focuses on who develops them, what their status means, and how an engineering team should evaluate one.

## Mental Model

Think of PHP-FIG as a standards collaboration with three distinct layers:

```text
PHP projects and community contributors
                  ↓ discuss shared problems
          Working Groups and governance
                  ↓ publish reviewed artifacts
        PSRs, PERs, and supporting resources
                  ↓ voluntary adoption
 Frameworks, libraries, applications, and tooling
```

The first layer brings implementation experience and use cases. Working Groups turn a problem into a proposal and test it through discussion and trial implementations. The resulting specification is an agreed contract that projects may adopt. A consumer may depend on an interface while a provider implements it, but both teams still need to understand the contract's semantics and the project's own compatibility policy.

## Purpose and Boundaries

PHP-FIG's mission is to advance the PHP ecosystem by bringing projects and people together to collaborate on standards. Its current bylaws describe three primary forms of output: PHP Standard Recommendations (PSRs), PHP Evolving Recommendations (PERs), and Auxiliary Resources (ARs). A PSR defines an interoperability contract between providers and consumers. A PER can capture practices, references, guidelines, or tooling intended to evolve with the language and ecosystem. An AR is a related tool, library, or example that supports a PSR or PER. The [mission and structure bylaws](https://www.php-fig.org/bylaws/mission-and-structure/) define these roles.

The word “standard” can sound more compulsory than it is. A PSR does not alter PHP syntax, the Zend Engine, or Composer. PHP does not automatically validate that an implementation conforms, and PHP-FIG membership does not require a project to implement any PSR. The group’s [FAQ](https://www.php-fig.org/faqs/) says that even voting projects are not obligated to implement accepted recommendations. The intended value is shared vocabulary and a common integration contract for projects that choose to use it.

A recommendation also has a scope. One specification may define method signatures, while another may set rules for names, values, error behavior, or transport-independent semantics. It does not necessarily answer every operational question around an implementation. For example, an interface with a `send()` method does not by itself specify retry policy, duplicate delivery, timeouts, or how success is observed. Teams must read the whole specification and then make their application-level decisions explicit.

## Who Participates

PHP-FIG governance is project-based. A Member Project is an established, publicly available PHP project or significant stakeholder organization. Each project selects an authorized Project Representative, who casts that project's formal votes under the current bylaws. The [mission and structure bylaws](https://www.php-fig.org/bylaws/mission-and-structure/) currently describe a twelve-member Core Committee that decides which specifications the group will consider and which proposals are approved. Secretaries administer votes and the group's communication and publication channels.

Formal project membership and community participation are different. You do not need to represent a Member Project to contribute feedback. PHP-FIG invites people to its public discussion channels, including its mailing list and Discord; the [Get Involved page](https://www.php-fig.org/get-involved/) explains how to join those discussions. Community members can raise use cases, review proposals, and contribute constructive feedback, but formal votes are limited to the roles specified in the voting bylaws.

Individuals may also apply to join a Working Group. The Editor considers whether the applicant's experience or perspective would help the work. Under the current bylaws, a full PSR Working Group includes an Editor, a Core Committee Sponsor, and at least three other people. The Editor coordinates the group's work and has final authority over its output during Draft. The Sponsor supports the process and must agree with the Editor that the proposal and meta document are ready before the Editor calls a Readiness Vote. During Review, the Sponsor becomes the final authority on changes and on moving the proposal forward or back; the Editor may veto changes they believe harm the specification's design. These roles are not badges of general PHP authority. They are responsibilities within a particular proposal's process.

If you want a proposal considered, start by describing an interoperability problem and the projects or users affected. Bring examples from working implementations, identify incompatible existing approaches, and discuss the costs of each proposed rule. A narrow problem statement and evidence make better starting points than a request to standardize a preferred library's API.

## How a PSR Is Developed

The formal PSR workflow has stages so that a proposal can attract feedback before it becomes a stable target for implementers:

1. **Pre-Draft:** Interested participants discuss a problem and outline an approach. A PSR needs a Full Working Group. Its Sponsor may ask the Core Committee to vote on whether PHP-FIG should consider the topic. If that Entrance Vote passes, the proposal receives a PSR number and enters Draft.
2. **Draft:** The Working Group develops the proposal in public discussion, pull requests, and other official channels. Alternatives can be explored, and major rewrites are possible. A meta document records important alternatives, trade-offs, and reasons for the choices made. When the Editor and Sponsor agree that the proposal is ready and the meta document is objective and complete, the Editor may call a Readiness Vote to decide whether it is ready for trials.
3. **Review:** The target is reasonably fixed so independent implementations can test whether it is practical. The current workflow requires at least four weeks in Review and two independent, viable trial implementations before an Acceptance Vote may be called. If those trials reveal that the proposal needs a major redesign, it returns to Draft.
4. **Accepted:** The Acceptance Vote makes the proposal an accepted PSR. The Working Group dissolves, and the Editor becomes the specification's Maintainer for errata and supporting artifacts.

The [PSR Workflow bylaw](https://www.php-fig.org/bylaws/psr-workflow/) spells out the stages and responsibilities. The [Voting Protocol](https://www.php-fig.org/bylaws/voting-protocol/) identifies who can vote at each stage: the Core Committee handles Entrance and Acceptance Votes, while Working Group members vote on Readiness. These governance details can change; consult the current bylaws when participating in a live proposal.

An accepted PSR is treated as a stable target. Its meaning and backward-compatibility contract cannot be changed through errata. Errata can clarify confusion in the meta document; a minimal pointer may be added to the specification itself. If substantive updates are needed or errata cannot resolve the confusion, the PSR must be replaced through the workflow, and the original may later be deprecated. A proposal shown as Draft or Review is still subject to change; the [PSR index](https://www.php-fig.org/psr/) publishes current statuses. Check the status and version of the exact document you intend to implement instead of relying on a blog post or an old implementation.

## What a PSR Does and Does Not Promise

Specifications often use capitalized terms such as `MUST`, `SHOULD`, and `MAY` to express requirement strength. PHP-FIG's FAQ explains that this convention follows [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119). Read those terms as normative language for the specified contract, not as requirements automatically enforced by the PHP interpreter.

An accepted PSR does not certify a package, prove that its implementation is correct, or guarantee that every framework has adopted it. An implementation can claim compatibility and still have bugs. Different versions of a standard can define different contracts. A PSR also does not decide your project's architectural choices: you can use an adapter, implement a competing local abstraction, or decline to adopt the recommendation if the trade-offs do not fit.

The group’s current site distinguishes PSRs from PERs and ARs. PERs are intended for recommendations that can evolve over time; PSRs target a particular stable interoperability contract. ARs supply related tools or examples. Do not treat all PHP-FIG publications as interchangeable or infer their authority from their appearance on the same website.

## Evaluate a Standard Before Adopting It

Use a concrete review rather than adopting a PSR because its number is familiar or because a framework includes it:

| Review question | Why it matters |
| --- | --- |
| What is the proposal's current status and version? | A Draft can change substantially; an Accepted document is the stable target. |
| Which provider-consumer boundary does it define? | A standard should solve a real integration problem in the codebase. |
| What behavior is normative beyond method signatures? | Error handling, mutability, value meaning, and edge cases often determine interoperability. |
| Which PHP versions and dependencies does the implementation support? | The standard's contract does not override runtime and package compatibility constraints. |
| What migration or adapter work will adoption require? | An interface boundary can still affect public APIs and deployment. |
| How will conformance be tested? | A shared contract needs tests that catch incompatible behavior, not only matching names. |

The decision is often not “standard or no standard.” A project can depend on a standard at its package boundary and retain an internal domain model behind an adapter. That limits coupling while allowing the public integration to interoperate. If the standard is broader than needed, keep the adapter small and document which parts of the contract the application relies on.

## A Contract at a PHP Package Boundary

The following fictional interface illustrates the distinction between a PHP type shape and its full behavioral contract. It is not a PSR:

```php
<?php

declare(strict_types=1);

interface NotificationSender
{
    /** @throws DeliveryFailed */
    public function send(string $recipient, string $message): void;
}

final class DeliveryFailed extends RuntimeException
{
}

final class InMemoryNotificationSender implements NotificationSender
{
    /** @var list<array{recipient: string, message: string}> */
    public array $sent = [];

    public function send(string $recipient, string $message): void
    {
        $this->sent[] = ['recipient' => $recipient, 'message' => $message];
    }
}

final class SignupService
{
    public function __construct(private NotificationSender $sender)
    {
    }

    public function welcome(string $email): void
    {
        $this->sender->send($email, 'Welcome');
    }
}

$sender = new InMemoryNotificationSender();
(new SignupService($sender))->welcome('reader@example.test');
var_export($sender->sent);
```

Any provider that implements the interface can be passed to `SignupService`, so the consumer does not depend on a specific mail vendor. But the PHP declaration leaves important questions open: does `send()` wait for remote delivery, queue the message, or merely accept it? Can it throw after the remote system has accepted a message? Are retries safe? A useful interoperability specification either defines the behavior necessary for providers and consumers to agree or clearly leaves those decisions to the application. An interface by itself is not enough to infer semantics.

When implementing a real standard, map its requirements to code intentionally. Use the required public names and signatures, preserve documented return and exception behavior, and do not add assumptions that the specification does not make. Hide provider-specific behavior behind adapters where it is not part of the contract. Avoid claiming broad conformance based only on matching class names.

## Testing and Operations

Tests should verify both sides of an integration. A consumer test should show how the application behaves with a conforming implementation. A provider should run tests that check the specified inputs, outputs, exceptions, and edge cases. If the specification leaves a choice open, test the chosen behavior at the application boundary and document it. When upstream libraries provide contract tests or interoperability suites, use them as evidence, then add application-specific tests for the cases they cannot know.

Review the exact dependency and standard version in Composer metadata and the lock file; [Chapter 94](094-composer-json.md) explains the manifest and [Chapter 95](095-composer-lock.md) explains the selected package set. During upgrades, check the standard's status, the package's compatibility notes, and whether the interface changed. A passing static analysis run verifies types and some structural expectations, but it does not prove semantic conformance.

Standards reduce integration friction; they do not remove operational failure. Network calls can time out, providers can partially complete work, and retries can duplicate effects. Apply ordinary production controls such as bounded timeouts, idempotency, observability, and explicit failure policy at the boundary. A package can be PSR-compatible and still be insecure, unreliable, or unsuitable for a particular workload.

## Common Mistakes

- Treating PHP-FIG as the PHP language standards committee or assuming a PSR changes PHP runtime behavior.
- Assuming acceptance forces every Member Project, framework, or package to implement the PSR.
- Treating a Draft or Review proposal as stable and building a public dependency on it without understanding its status.
- Reading only interface declarations while ignoring normative prose, edge cases, and intentionally unspecified behavior.
- Believing a type-compatible implementation is automatically behaviorally interoperable.
- Adopting a standard without checking its current version, PHP requirements, package maintenance, and migration cost.
- Conflating formal voting membership with public discussion and contribution opportunities.
- Assuming a standard guarantees security, performance, or operational behavior that it does not define.

## Senior Engineer Thinking

The useful question is not “Do we follow every PSR?” It is “Which stable contracts reduce friction at the boundaries we actually share, and what local behavior remains our responsibility?” A standard can improve collaboration across independently released packages, but every external contract adds a compatibility surface that must be maintained and tested.

Read the primary specification, check its lifecycle status, identify the provider and consumer, and write down the decisions it leaves open. Then choose the smallest integration seam that captures its value. That practice respects the work of a standards group while keeping the application accountable for its own architecture and operations.

## Exercises

1. Choose a package boundary in an application where two independently maintained components exchange objects. State the interoperability problem and identify which behavior would need to be shared for an interface alone to be insufficient.
2. Visit a current Draft or Review proposal in the PHP-FIG index. Record its status, Editor, Sponsor, and the remaining work described by its meta document. Explain what could still change.
3. Find an accepted PSR relevant to a package you use. List its normative requirements separately from behavior it leaves to implementers. Design one provider-side and one consumer-side contract test.
4. Propose a small change to a shared PHP convention. Write a problem statement, affected stakeholders, two alternative approaches, and one risk that a trial implementation should test.
5. Your framework implements an accepted PSR, but your application depends on a package that only partially conforms. Describe how you would verify the mismatch and isolate it behind an adapter.
6. Explain why subscribing to public discussions can influence a proposal but does not by itself grant formal voting rights under the bylaws.

## Review Questions

1. What problem does PHP-FIG aim to solve for independent PHP projects?
2. How do a PSR, a PER, and an Auxiliary Resource differ at a high level?
3. Who represents a Member Project in formal voting, and how can a non-member participate in discussion?
4. What is the role of a Working Group's Editor and Sponsor?
5. Why does a proposal move through Draft and Review before it can be accepted?
6. What does an accepted status mean, and what does it not guarantee?
7. Why are normative words in a PSR different from instructions enforced by the PHP language runtime?
8. What should a team inspect before adopting a standard at a package boundary?
9. Why can matching method signatures still fail to provide behavioral interoperability?
10. Which production concerns remain the responsibility of an implementation even when it follows a PSR?

## Summary

PHP-FIG is a collaboration among PHP projects and contributors to develop interoperability standards and related resources. Its governance is project-based: Member Projects select Project Representatives for formal votes, while community members can contribute through public discussion and may apply to Working Groups. PSRs move through Pre-Draft, Draft, Review, and Accepted stages, with public feedback and trial implementations helping evaluate the contract.

A PSR defines a provider-consumer interoperability contract, but it does not change PHP itself, force adoption, certify implementations, or guarantee security and operational behavior. Check the exact status and version, read normative text beyond signatures, determine whether the contract solves a real boundary problem, and test both provider and consumer behavior. Chapter 100 catalogs the standards; Chapter 98 contains the detailed PSR-4 mapping rules.

## References

- [PHP-FIG mission and structure bylaws](https://www.php-fig.org/bylaws/mission-and-structure/)
- [PHP-FIG PSR workflow](https://www.php-fig.org/bylaws/psr-workflow/)
- [PHP-FIG PSR amendments](https://www.php-fig.org/bylaws/psr-amendments/)
- [PHP-FIG voting protocol](https://www.php-fig.org/bylaws/voting-protocol/)
- [PHP-FIG: Get Involved](https://www.php-fig.org/get-involved/)
- [PHP-FIG: Frequently Asked Questions](https://www.php-fig.org/faqs/)
- [PHP-FIG: PSR index and statuses](https://www.php-fig.org/psr/)
- [RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)
- [Chapter 97 — Autoloading](097-autoloading.md)
- [Chapter 98 — PSR-4](098-psr-4.md)
- [Chapter 100 — PSR Standards](100-psr-standards.md)
