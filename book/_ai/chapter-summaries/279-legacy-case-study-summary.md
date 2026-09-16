# AI Summary — Chapter 279 — Legacy Case Study

- Status: complete
- Volume: Volume XVIII — LEGACY PHP
- Last updated: 2026-09-16

## Written material

Chapter 279 presents the fictional Northstar Parts order portal, a PHP 5.6.40 application with shared bootstrap code, global database access, web pages, CLI/cron jobs, reporting consumers, and a warehouse integration. The case study shows how to define a migration charter, inventory hidden contracts, characterize behavior, choose a read-only seam, migrate runtime and schema in separate dimensions, own an ambiguous external effect, roll out by failure domain, and use forward recovery when binary rollback is unsafe.

## Concepts already explained

Legacy migration charter, fictional case facts versus engineering principles, hidden-contract inventory, unknowns as owned risks, characterization fixtures, normalized observations, compatibility adapter, read-only migration seam, runtime/schema/effect separation, nullable schema expansion, export operation identity, unknown external outcome, reconciliation, forward repair, failure-domain rollout, stop conditions, rollback decision tree, and seam retirement conditions.

## Terminology established

Northstar Parts, order portal, staff order lookup, warehouse export, compatibility bootstrap, `export_operation_id`, export operation identity, `ExportOutcome`, `ExportAttempt`, and unknown warehouse outcome.

## Examples used

- A PHP 5.6.40 Northstar Parts order portal with Apache/mod_php, shared bootstrap, global MySQL access, staff lookup, and nightly export.
- A migration charter separating capability, runtime, data, effect, ownership, evidence, and rollback concerns.
- An inventory table covering bootstrap, order state, money, export, authorization, runtime, and reporting consumers.
- Characterization fixtures exposing an export-marking bug after a warehouse failure.
- A PHP 5 legacy lookup page and a PHP 8.2+ typed `OrderReader` compatibility adapter.
- A runtime compatibility matrix and a nullable `export_operation_id` expand-and-contract sequence.
- Typed `ExportOutcome` and `ExportAttempt` values for accepted, rejected, and unknown provider results.
- A rollout stop-condition list and a code/data/effect rollback decision tree.

## Cross-references

The chapter links to Chapters 270–278 for the legacy-PHP migration techniques it combines, Chapters 241–243 for partial failure, idempotency, and message delivery, and Chapters 264–265 for deployment and rollback.

## Open threads

Begin Volume XIX with Chapter 280 — Tennis Reservation Service, carrying the case-study discipline into a small project with explicit state, time windows, conflict rules, and a bounded reservation workflow.

## Exact next section

Chapter 280 — Tennis Reservation Service: the Why This Matters section.

## Technical verification notes

The PHP 8.2+ typed examples use enums and readonly classes and should be linted with PHP 8.2 or newer. The PHP 5 snippet is illustrative legacy source and was not claimed to run on the modern interpreter. Local Markdown links and `git diff --check` should be run after each writing session.

## Writing notes

Keep the case study continuous and distinguish fictional facts from general engineering principles. Preserve the established concerns around authorization, integer money, durable operation identity, post-commit effects, compatibility windows, and forward recovery as later small projects introduce their own domains.
