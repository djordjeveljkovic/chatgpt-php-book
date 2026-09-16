# AI Summary — Chapter 99 — PHP-FIG

- Status: complete
- Volume: Volume 7 — COMPOSER AND THE PHP ECOSYSTEM
- Last updated: 2026-09-15

## Written material

Explains PHP-FIG's interoperability mission, organizational boundaries, current project-based governance, community participation, PSR lifecycle, distinctions among PSRs/PERs/ARs, how to evaluate and adopt a standard, contract semantics beyond PHP types, testing, operations, exercises, and review questions. Leaves the detailed PSR catalog to Chapter 100 and PSR-4 rules to Chapter 98.

## Concepts already explained

- PHP-FIG coordinates project-level interoperability work; it does not alter PHP runtime behavior or compel projects to adopt a PSR.
- Member Projects designate Project Representatives for formal governance. Public discussion and Working Group participation are separate contribution paths.
- PSRs advance through Pre-Draft, Draft, Review, and Accepted stages; the Editor and Sponsor agree before Readiness, and during Review the Sponsor controls changes and progression subject to an Editor veto.
- An accepted PSR keeps its meaning and backward-compatibility contract. Errata may add backward-compatible clarification to the meta document; substantive changes require a replacement PSR.
- PSRs target stable interoperability contracts; PERs are evolving recommendations and ARs support PSRs/PERs with related tools, code, or examples.
- Method signatures alone may leave semantics such as retries, timing, and failure behavior unspecified; implementations must test and document their boundary behavior.
- Adoption should be based on the document's exact status, version, normative requirements, compatibility cost, and fit with a real package boundary.

## Terminology established

PHP-FIG, Member Project, Project Representative, Core Committee, Secretary, Working Group, Editor, Sponsor, PSR, PER, Auxiliary Resource (AR), Pre-Draft, Draft, Review, Accepted, trial implementation, provider, consumer, normative requirement, conformance.

## Examples used

- A conceptual flow from projects/community through Working Groups and governance to published artifacts and optional adoption.
- A fictional `NotificationSender` PHP interface illustrating that method shape does not fully specify delivery, retries, or failure semantics; explicitly not a PSR.
- Evaluation questions, contract-testing approach, and a proposal problem-statement exercise.

## Cross-references

- [Chapter 97 — Autoloading](../../volumes/07-composer-and-the-php-ecosystem/097-autoloading.md)
- [Chapter 98 — PSR-4](../../volumes/07-composer-and-the-php-ecosystem/098-psr-4.md)
- [Chapter 100 — PSR Standards](../../volumes/07-composer-and-the-php-ecosystem/100-psr-standards.md)
- PHP-FIG official mission/bylaws, workflow, amendments, voting, participation, FAQ, and status index are cited in the chapter.

## Open threads

No open chapter work remains. Current governance/process details should be rechecked against PHP-FIG bylaws if the chapter is revised after September 2026.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

Current PHP-FIG mission/structure, member-project roles, Working Group rules, PSR lifecycle, amendment/errata boundaries, status descriptions, formal vote roles, and participation channels were checked against official PHP-FIG bylaws, Get Involved, FAQ, and PSR index on 2026-09-15. Independent proofreading corrected the Readiness/Review authority descriptions and the limits on accepted-PSR errata. Local links resolved, the PHP example passed lint and runtime execution, and `git diff --check` passed.

## Writing notes

Do not duplicate the PSR catalog in Chapter 100 or the exact PSR-4 namespace-to-path rules in Chapter 98. Treat governance details as current bylaws, not permanent facts.
