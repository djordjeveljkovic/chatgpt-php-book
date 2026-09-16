---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 279
title: Legacy Case Study
slug: legacy-case-study
status: complete
summary: ../../_ai/chapter-summaries/279-legacy-case-study-summary.md
---

# Chapter 279 — Legacy Case Study

## Why This Matters

The preceding chapters described techniques for understanding and changing a legacy PHP system. A case study makes the techniques interact. In a real migration, characterization tests, a route seam, a database transition, and a PHP upgrade are not separate assignments. They constrain one another.

The system in this chapter is deliberately ordinary: a PHP 5 order portal that grew around shared includes, a database, scheduled scripts, and a few integrations. Its difficulty comes from invisible contracts, not unusual algorithms. The goal is to show how an engineer turns an intimidating application into a sequence of owned decisions with evidence, stopping points, and recovery options.

## The System We Inherited

Northstar Parts sells replacement components to business customers. Its portal has three important capabilities:

* customers search the catalog and submit orders;
* staff review orders and change their status;
* a nightly job exports paid orders to a warehouse system.

The application runs PHP 5.6.40 behind Apache with mod\_php. A shared `bootstrap.php` reads configuration, starts sessions, opens a global `mysql` connection, defines helper functions, and installs an error handler. Pages include that file directly. The database is MySQL, and the order table is also read by a reporting script and an ad hoc support tool.

The order path is approximately:

~~~text
POST /order.php
  → bootstrap.php
  → $_POST validation
  → global $db query
  → orders.state = 'paid'
  → echo confirmation HTML

nightly export.php
  → bootstrap.php
  → SELECT paid orders
  → write CSV
  → mark orders.exported = 1
~~~

The business wants a supported PHP release, a safer deployment process, and an API for a new warehouse integration. It cannot pause order entry for a multi-month rewrite. That constraint determines the migration shape.

## First Decision: State the Boundaries

The team writes a migration charter before selecting a framework or rewriting a page:

~~~text
capability: order submission and warehouse export
preserve: authorization, cents precision, order state transitions, export meaning
first seam: read-only staff order lookup
source runtime: PHP 5.6.40, Apache/mod_php, CLI cron
target runtime: PHP 8.3 FPM and CLI, exact image recorded in CI
data rule: paid orders are immutable except for export metadata
effect rule: one warehouse export per order operation
rollback: route and artifact rollback while old schema/messages remain readable
stop: unexplained authorization, money, duplicate-export, or data-loss difference
owners: application, database, operations, and warehouse integration
~~~

This charter prevents “modernization” from becoming permission to change every contract at once. The first capability is intentionally read-only. A read-only slice gives the team a useful seam without immediately combining authorization, money mutation, and an external effect.

## Inventory the Hidden Contracts

The team searches source, deployment files, cron entries, SQL, templates, and logs. It records facts and unknowns separately:

| Surface | Observation | Risk | Next evidence |
| --- | --- | --- | --- |
| bootstrap | starts sessions and opens a global connection | duplicate initialization in FPM | request identity test |
| order state | pages write `pending`, `paid`, and `cancelled` | conflicting writers | query and source inventory |
| money | `total` is a decimal column and is formatted in PHP | rounding drift | fixture comparison |
| export | CSV is written before the row is marked exported | ambiguous retry | trace one interrupted run |
| auth | staff page checks a session role inline | route seam could bypass policy | authorization matrix |
| runtime | cron uses a different PHP binary | false compatibility confidence | identity check per entry point |
| reporting | a support script reads `orders.*` | schema contraction may break it | script inventory |

An unknown is not an empty cell. It is a risk with an owner. The support script is especially important: it is not in the web deployment, but it is still a consumer of the database contract.

## Characterize Before Refactoring

The team chooses representative orders rather than starting with a code formatter. Fixtures include a normal paid order, a zero-quantity attempt, a decimal total, a missing customer, an unauthorized staff request, and a repeated export. For each fixture it records:

* response status, redirect, relevant headers, and normalized body;
* database rows changed and exact monetary values;
* audit and export files produced;
* warnings, logs, and exit codes;
* authorization result and actor identity;
* behavior after a retry or worker interruption.

The first surprising observation is that a failed warehouse write still sets `exported = 1` because the script marks the row after opening the file, not after the warehouse acknowledgment. That is a business bug and a migration constraint. A new implementation must not silently “fix” it without a repair and duplicate-export policy.

## Find a Narrow Seam

The first extracted capability is staff order lookup. The old page has mixed concerns:

~~~php
<?php

require_once __DIR__ . '/bootstrap.php';

if (!isset($_SESSION['staff_role']) || $_SESSION['staff_role'] !== 'support') {
    header('HTTP/1.1 403 Forbidden');
    exit('Forbidden');
}

$id = (int) (isset($_GET['id']) ? $_GET['id'] : 0);
$result = mysql_query("SELECT id, state, total FROM orders WHERE id = $id", $db);
$order = mysql_fetch_assoc($result);
echo render_order($order);
~~~

It performs authentication, input conversion, SQL construction, data access, and rendering in one script. The first seam preserves the observable contract while moving ownership of the lookup behind a small adapter:

~~~php
<?php

declare(strict_types=1);

final readonly class OrderView
{
    public function __construct(
        public int $id,
        public string $state,
        public string $total,
    ) {
    }
}

interface OrderReader
{
    public function findForStaff(int $id): ?OrderView;
}

final readonly class LegacyOrderReader implements OrderReader
{
    public function __construct(private LegacyDatabase $database)
    {
    }

    public function findForStaff(int $id): ?OrderView
    {
        if ($id < 1) {
            return null;
        }

        $row = $this->database->fetchOrder($id);

        return $row === null
            ? null
            : new OrderView(
                (int) $row['id'],
                (string) $row['state'],
                (string) $row['total'],
            );
    }
}
~~~

The example adapter is not a complete database implementation. Its value is the boundary: rendering and routing can be tested against `OrderReader`, while the legacy query remains replaceable. The adapter must still preserve authorization; dependency injection does not grant access by itself.

## Move the Runtime in a Separate Dimension

The team does not change the database schema, framework, and PHP binary in one release. It first builds the application under PHP 8.3, runs the compatibility scan, and compares the characterization suite. The compatibility matrix includes web, CLI, and cron because the old cron entry still points to `/usr/bin/php5`.

The first target artifact contains a compatibility bootstrap and a PDO-based database adapter. It does not yet own order writes. The team records differences explicitly:

| Difference | Classification | Decision |
| --- | --- | --- |
| undefined array key warning is logged | intentional observability change | fix caller; do not hide warning |
| decimal total rendered with a changed trailing zero | behavior difference | block until money contract is settled |
| missing `mysql` extension | required runtime change | use tested PDO adapter |
| old session cookie unreadable | rollout risk | preserve cookie/session format or fence cohort |
| export retry remains ambiguous | pre-existing data/effect risk | block export cutover, repair first |

“The target starts” is therefore one green check among several. The team can deploy the read-only staff lookup only after authorization, output, database reads, and rollback have evidence.

## Make the Database Transition Compatible

To support the new warehouse integration, the team adds `export_operation_id` as nullable data. It does not drop `exported`, rename `state`, or add a uniqueness constraint before measuring existing duplicates.

The sequence is:

1. add the nullable operation field;
2. deploy readers that tolerate a missing operation ID;
3. inventory and reconcile rows already marked exported;
4. write the operation ID with the export claim in one transaction where possible;
5. backfill only rows with a known, auditable operation identity;
6. compare old and new export observations;
7. switch one warehouse cohort to the new path;
8. contract old metadata only after reports, support tools, and rollback artifacts have moved.

The old export bug changes the plan. A row marked exported without an acknowledgment cannot be guessed into a successful or failed category. It enters a review queue with evidence from the file, warehouse response, logs, and operation history. Forward repair is safer than declaring every such row unexported and risking duplicate shipment.

## Own the External Effect

The warehouse boundary is an effect boundary. The new exporter records an operation identity before sending, uses a stable payload, and treats an ambiguous timeout as “unknown” rather than “failed and safe to retry.” A retry first asks the warehouse or reconciliation process whether the operation was accepted.

~~~php
<?php

declare(strict_types=1);

enum ExportOutcome: string
{
    case Accepted = 'accepted';
    case Rejected = 'rejected';
    case Unknown = 'unknown';
}

final readonly class ExportAttempt
{
    public function __construct(
        public string $operationId,
        public ExportOutcome $outcome,
        public string $evidence,
    ) {
        if ($operationId === '' || $evidence === '') {
            throw new InvalidArgumentException('Export attempt is incomplete');
        }
    }
}
~~~

The enum makes the ambiguous state visible to callers. It does not make the warehouse idempotent. That guarantee belongs at the integration contract, where the team must confirm the operation key, payload, retry, and reconciliation semantics.

## Roll Out by Failure Domain

The rollout has separate switches for the staff lookup, order writes, and warehouse export. Web traffic moves first for a low-risk internal cohort. CLI repair commands and cron jobs move only after their binary, configuration, working directory, permissions, and lock behavior are verified. Queue or integration workers are drained and upgraded with message compatibility checks.

Useful stop conditions include:

* any unexplained authorization or tenant-isolation difference;
* a mismatch in cents, tax, or order-state transitions;
* duplicate or unknown warehouse operations above the agreed threshold;
* a rise in database lock waits, queue age, or worker memory;
* an old artifact that can no longer read the resulting schema or messages;
* an unowned compatibility finding in a path that may mutate data.

Rollback is a decision tree, not a button:

~~~text
route or cohort regression?
  ├─ no durable change → switch traffic to the prior implementation
  ├─ compatible durable change → roll back code, keep schema, reconcile
  └─ external effect or ambiguous commit → fence retries, investigate, forward-repair
~~~

The old application remains available only while its reads, writes, sessions, and messages remain compatible. A binary rollback cannot undo a warehouse shipment or safely reinterpret a newly written status.

## What the Team Actually Achieved

After three iterations, Northstar has not rewritten the whole application. It has achieved something more valuable:

* staff lookup has one explicit reader boundary and characterization coverage;
* PHP 8.3 web and CLI artifacts have separate runtime evidence;
* order writes still have one named authority rather than competing implementations;
* warehouse export has an operation identity and an unknown outcome state;
* schema expansion, repair, rollout, and rollback have owners and stop conditions;
* the remaining legacy pages are now an inventory of bounded capabilities, not an undifferentiated rewrite.

The case study demonstrates sequencing. Each change creates evidence for the next change. None of the seams is automatically permanent: every adapter, dual field, feature flag, and compatibility branch needs an owner and a removal or review condition.

## Common Mistakes

* Starting with a framework rewrite before identifying the business and recovery contracts.
* Calling an undocumented support script “out of scope” because it is not in the web repository.
* Treating a PHP 8 build as proof that PHP 5 behavior has been preserved.
* Changing the export bug during a runtime migration without a duplicate-effect policy.
* Giving the new reader authority over writes merely because it can read the same table.
* Adding a unique constraint before measuring duplicate and unknown operation identities.
* Testing only browser traffic while cron and CLI still use the old binary.
* Treating an ambiguous warehouse timeout as a safe failure.
* Keeping route switches and adapters without recording their owner, evidence, or expiry.
* Rolling back code after an external effect without fencing retries and reconciling state.

## Senior Engineer Thinking

The senior question is not “how do we modernize this old PHP application?” It is “which capability can we make explicit first, what contract must remain stable, what evidence distinguishes a safe difference from a regression, and which irreversible effects need forward recovery?”

Legacy work becomes tractable when it is decomposed by ownership and evidence. Start with inventory, characterize behavior, choose a narrow seam, separate runtime and schema changes where possible, make effects idempotent or observable, roll out by failure domain, and keep the old path only as long as its compatibility is understood.

## Exercises

1. Extend the Northstar inventory with session cookies, report queries, backups, and customer-support repair actions. Mark each unknown and assign an owner.
2. Write characterization fixtures for a paid order, a repeated submission, a decimal total, and an interrupted warehouse export. State the normalized observations.
3. Design a compatibility adapter for the old order lookup. List the preserved behavior, changed behavior, authorization check, and removal condition.
4. Create an expand-and-contract plan for `orders.state` becoming `orders.status`. Include old writers, dual reads, conflict repair, and rollback.
5. Draw the recovery path after a warehouse timeout where the request may have committed locally but the external result is unknown.

## Review Questions

* Why is this case study sequenced around a read-only seam first?
* Which hidden contracts did the inventory reveal?
* What did characterization testing discover about the old export path?
* Why does a compatibility adapter not replace authorization?
* Why are runtime, schema, and external-effect changes separated?
* What does an unknown warehouse outcome require before retry?
* Which rollout stop conditions protect data and business effects?
* When is route rollback safe, and when is forward repair required?
* Why must CLI and cron be verified separately from web traffic?
* What makes a migration seam temporary and accountable?

## Summary

The Northstar case study combines legacy inventory, characterization tests, narrow seams, PHP runtime migration, compatible schema expansion, external-effect ownership, staged rollout, and forward recovery. The team does not rewrite everything. It makes one capability explicit, preserves and measures its contracts, separates risky dimensions, and expands only when evidence and rollback are real. That is the practical discipline for changing a legacy PHP system without losing control of it.

## References

- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](./271-legacy-architecture.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 273 — Safe Refactoring](./273-safe-refactoring.md)
- [Chapter 274 — Strangler Pattern](./274-strangler-pattern.md)
- [Chapter 275 — Branch by Abstraction](./275-branch-by-abstraction.md)
- [Chapter 276 — Framework Migration](./276-framework-migration.md)
- [Chapter 277 — Database Migration](./277-database-migration.md)
- [Chapter 278 — PHP Version Migration](./278-php-version-migration.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../16-distributed-systems/242-idempotency.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
