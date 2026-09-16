---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 175
title: Testing Legacy Code
slug: testing-legacy-code
status: complete
summary: ../../_ai/chapter-summaries/175-testing-legacy-code-summary.md
---

# Chapter 175 — Testing Legacy Code

## Why This Matters

Legacy code is code whose behavior is poorly understood, difficult to change safely, or constrained by an old runtime, dependency, data model, or deployment process. It may be valuable and reliable in production while lacking tests. The immediate goal is to make a safe change possible, not to rewrite everything or reach an arbitrary coverage number.

Testing legacy code requires a different starting point. First observe the current behavior, record the behavior that must remain, and introduce seams where a test can control an external dependency. Then make one small change and let the tests provide a safety signal.

## Start with Risk and Characterization

Inventory the code path, users, data, external effects, failure history, and planned change. Rank behavior by business and safety risk: money, authorization, data migration, notifications, and destructive operations deserve stronger characterization than dead administrative screens.

A characterization test captures what the system currently does. It is not automatically a statement that every current behavior is desirable. Name it as observed behavior, review the result with a domain owner, and then decide which behavior is a requirement, a compatibility constraint, or a bug to change deliberately.

~~~php
<?php

declare(strict_types=1);

final class LegacyPrice
{
    public function total(string $quantity, string $unitPrice): string
    {
        // Existing behavior is intentionally simple for this example.
        return number_format((float) $quantity * (float) $unitPrice, 2, '.', '');
    }
}

function testLegacyPriceCharacterization(): void
{
    $calculator = new LegacyPrice();

    assert($calculator->total('2', '10.50') === '21.00');
    assert($calculator->total('0', '10.50') === '0.00');
}
~~~

Run characterization tests against a representative fixture set, including malformed and boundary inputs that real users have produced. Store selected outputs as explicit assertions rather than an opaque snapshot when the output needs review. Snapshot or golden-master tests can be useful for a large report, but make the diff reviewable and redact sensitive data.

## Find Seams

A seam is a place where behavior can be changed or a dependency replaced without editing every caller. Useful seams include a function parameter, an object interface, a wrapper around a global, a command boundary, a database repository, a filesystem adapter, or a process invocation.

If a legacy function reads the clock and sends email directly, add a narrow wrapper first. The wrapper preserves behavior while giving later tests a controlled boundary.

~~~php
<?php

declare(strict_types=1);

interface LegacyMailer
{
    public function send(string $recipient, string $subject, string $body): void;
}

final class InvoiceReminder
{
    public function __construct(private LegacyMailer $mailer)
    {
    }

    public function remind(string $email, int $daysOverdue): void
    {
        if ($daysOverdue < 7) {
            return;
        }

        $this->mailer->send(
            $email,
            'Invoice reminder',
            'Your invoice is overdue.',
        );
    }
}
~~~

The first refactor need not introduce a complete architecture. It should reduce one source of uncertainty while preserving observable behavior. Keep the seam small enough that a fake can represent it and the production adapter remains obvious.

## Global State and Static Calls

Globals, static methods, superglobals, and ambient environment values make tests order-dependent. Replace them incrementally:

1. Add a wrapper with the current behavior.
2. Route one caller through the wrapper.
3. Inject the wrapper or an interface.
4. Add a test for the behavior and failure path.
5. Remove direct uses as callers migrate.

Do not globally reset every static value in a test teardown and call the problem solved. That can hide which code owns the state and can fail when parallel workers share a process or external resource. Make ownership explicit at the boundary.

For database calls, introduce a repository around one use case before attempting to replace a whole ORM or query layer. Test the repository against the real database and the service against a focused fake where that distinction helps.

## Control External Effects

Legacy code may combine validation, database writes, emails, filesystem changes, and redirects in one function. Start with a test-safe environment: a temporary database or schema, a mail sink, a temporary filesystem, and a fake clock. Do not replace every effect with a mock before learning which effects are required.

For destructive operations, add a dry-run or transaction boundary only when the production semantics support it. A test that rolls back a transaction cannot prove behavior performed through a separate connection or an external provider. Capture outgoing messages and assert their meaningful content, then use an integration test for the adapter.

## Small Refactoring Steps

Prefer a sequence that leaves the system runnable:

- add a characterization test;
- extract a pure calculation;
- introduce a parameter or interface;
- move one side effect behind an adapter;
- separate validation from a write;
- add a guarded database condition;
- rename only after tests describe the behavior;
- delete unreachable code after evidence and review.

Commit each safe step separately. If a characterization test fails after a refactor, decide whether the change is an intended fix or an accidental behavior change. Do not weaken the assertion merely to make the build green.

A strangler approach can place a new implementation behind the old entry point, route one cohort or operation to it, compare outcomes, and expand gradually. Keep the old path available only while it provides a real rollback or migration benefit; two implementations increase drift and test cost.

## Working with Missing Tests

When the full application cannot boot in a test process, begin at a smaller boundary: a parser, command handler, repository, CLI action, or HTTP endpoint with controlled infrastructure. Record the setup needed and remove unrelated global initialization from the path one dependency at a time.

Use static analysis, compiler errors, logs, and production metrics as additional evidence. They do not replace behavior tests, but they reveal unreachable branches, dynamic calls, type assumptions, and high-risk paths that deserve characterization. Add tests where a change is planned instead of attempting to document an entire legacy system before any improvement.

## Data and Compatibility

Legacy systems often encode meaning in database columns, status strings, date formats, or files consumed by other systems. Treat these as contracts until proven otherwise. Characterize migrations, rounding, timezone conversion, encoding, and unknown status behavior. Use copies of realistic shape with synthetic values and keep personal data out of fixtures.

When changing a schema, use an expand-and-contract sequence: add compatible structure, deploy code that reads both forms, backfill and verify, switch writes, then remove the old path after evidence. Test rollback and mixed-version behavior if old and new application processes can run together.

## Team and Delivery

Pair with a domain expert when observed behavior is ambiguous. Record why a test exists and which behavior it protects. Keep tests close to the changed boundary and measure feedback time. A slow legacy suite may need a fast characterization subset on every change and a broader integration run on a schedule or release gate.

Do not hide legacy risk behind a large refactor. An explicit known limitation with a regression test is safer than an unreviewed rewrite. As seams improve, move behavior tests down to faster layers and retain only the end-to-end journeys that protect real workflows.

## Common Mistakes

- Rewriting the system before understanding its behavior.
- Treating every current result as a desired requirement.
- Adding mocks everywhere without testing real adapters.
- Resetting global state blindly and hiding ownership.
- Using production data in fixtures.
- Making a broad refactor with no intermediate runnable state.
- Removing the old path before migration and rollback evidence exists.
- Chasing coverage numbers instead of risk and feedback.

## Senior Engineer Thinking

Legacy testing is controlled discovery. Characterize important behavior, create seams around uncertainty, make one reversible change at a time, and use integration tests where fakes cannot represent database or external semantics. The result is a safer path for future changes, even when the old design remains in service.

## Exercises

1. Choose a legacy function that mixes validation, a database write, and email. Write characterization cases and identify two seams.
2. Wrap a global clock or mail function behind an interface without changing its external behavior.
3. Design an expand-and-contract migration and test a mixed-version deployment.
4. Select a high-risk path where end-to-end coverage is justified and a lower-level path where a unit or integration test gives faster feedback.

## Review Questions

1. What is a characterization test, and why is it not automatically a requirements test?
2. What makes a useful seam in legacy code?
3. Why can a transaction rollback fail to isolate an external side effect?
4. How should a team handle behavior that is observed but not desired?
5. Why are small runnable refactoring steps safer than a broad rewrite?
6. When should a legacy path remain available during migration?

## Summary

Testing legacy code starts with risk-aware characterization, not a rewrite. Capture important behavior, create seams around globals and external effects, control test data and resources, refactor in small runnable steps, and use real integration tests where fakes cannot model database or provider semantics. Treat compatibility, migration, and rollback behavior as explicit contracts.

## References

- [Michael Feathers: Working Effectively with Legacy Code](https://www.informit.com/store/working-effectively-with-legacy-code-9780131177055)
- [Martin Fowler: Legacy Seams](https://martinfowler.com/bliki/LegacySeam.html)
- [PHPUnit documentation](https://docs.phpunit.de/)
- [PHP manual: Dependency injection concepts](https://www.php.net/manual/en/language.oop5.decon.php)
- [OWASP Legacy Application Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Legacy_Application_Management_Cheat_Sheet.html)

