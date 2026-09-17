---
book: The Complete Modern PHP Engineering Book
volume: 19
volume_title: SMALL ENGINEERING PROJECTS
chapter: 283
title: File Importer
slug: file-importer
status: complete
summary: ../../_ai/chapter-summaries/283-file-importer-summary.md
---

# Chapter 283 — File Importer

## Why This Matters

File import is often described as “accept a CSV and insert the rows.” That description hides the dangerous parts. A file is untrusted input with a size, encoding, format, storage, retention, and lifecycle. Its rows may be malformed, duplicated, reordered, or partially valid. An import may run longer than an HTTP request, consume more memory than a PHP-FPM worker, and fail after changing durable state.

This project builds an importer for a tenant’s product catalog. It demonstrates a safe file boundary, quarantine before parsing, streaming and bounded work, explicit row validation, deterministic duplicate rules, resumable jobs, idempotent writes, progress evidence, and recovery after partial failure.

## Define the Import Contract

Write the contract before choosing a parser:

~~~text
input: UTF-8 CSV with required headers sku,name,price_cents,active
owner: authenticated tenant may submit and inspect only its imports
limits: 50 MiB upload, 200 MiB stored file, 500,000 rows; archives unsupported
storage: random server-side object key in quarantine storage
processing: asynchronous job; upload request never imports rows
validation: file/header/schema/row errors are classified separately
duplicate SKU within the file: reject the later row and continue; existing tenant SKU: deterministic upsert
write policy: valid rows upsert by tenant_id + sku in bounded transactions
retry: job cursor and row outcome are durable; retries are resumable
retention: source and report have explicit deletion dates
effect: completion notification is emitted after import state commits
~~~

Decide whether one invalid row rejects the file, produces a partial import, or enters a review state. The example allows valid rows to continue but does not hide failures: the final report contains counts and bounded row-level diagnostics. A financial or regulatory import may require all-or-nothing behavior instead.

## Secure the Upload Boundary

Authenticate the tenant and authorize the import capability before accepting bytes. Enforce request size at the web server, PHP configuration, application, and storage layers; the smallest limit should be intentional and observable. Do not trust the client filename, extension, MIME type, or path.

The upload flow is:

~~~text
request → size/type checks → temporary upload
        → malware/content scan where required
        → quarantine object with random key
        → immutable import record
        → asynchronous processing job
~~~

Store outside the executable document root with a generated object key. Reject path separators, control characters, and user-provided storage paths. Use least-privilege storage credentials. If archives are supported, bound decompressed size, file count, nesting, and destination paths; an archive can be small on upload and enormous after expansion.

Do not parse directly from a user-controlled temporary path after the request ends. Move or copy the verified object into controlled storage and record its content hash, byte count, media policy, tenant, actor, and scan result. A hash identifies bytes; it does not prove the file is safe or that it has the expected schema.

## Inspect Before Parsing

Classify the file in stages:

1. transport: upload succeeded and is within limits;
2. storage: object exists, is readable, and matches recorded size/hash;
3. content: encoding, delimiter, line endings, and format are supported;
4. schema: required headers and column count are present;
5. rows: each value satisfies domain rules;
6. persistence: valid rows are written under the import’s ownership and transaction policy.

Do not infer CSV safety from a `.csv` suffix. CSV can contain formulas that become dangerous when opened in a spreadsheet. If exported reports are a later output, prefix or encode cells that begin with formula characters according to the product’s spreadsheet-injection policy. Do not “fix” source values silently without recording the transformation.

## Stream the Rows

Never load a 200 MiB file into one PHP string or array merely because the parser API makes that convenient. Stream rows, bound row and field sizes, and keep only the state needed for the current batch and progress record.

~~~php
<?php

declare(strict_types=1);

/** @return Generator<int, array<string, string>, void, void> */
function readCatalogRows(string $path): Generator
{
    $handle = fopen($path, 'rb');
    if ($handle === false) {
        throw new RuntimeException('Import file cannot be opened');
    }

    try {
        $header = fgetcsv($handle, null, ',', '"', '\\');
        if (is_array($header) && isset($header[0])) {
            $header[0] = preg_replace('/^\xEF\xBB\xBF/', '', $header[0]) ?? $header[0];
        }

        if ($header === false || $header !== ['sku', 'name', 'price_cents', 'active']) {
            throw new InvalidArgumentException('Unexpected catalog header');
        }

        $rowNumber = 1;
        while (($row = fgetcsv($handle, null, ',', '"', '\\')) !== false) {
            $rowNumber++;
            if (count($row) !== count($header)) {
                throw new InvalidArgumentException("Unexpected column count at row $rowNumber");
            }

            yield $rowNumber => array_combine($header, $row);
        }
    } finally {
        fclose($handle);
    }
}
~~~

The generator bounds the number of rows held by the caller, but it does not by itself bound a single field or a malformed line. Enforce maximum row bytes at the upload/content layer and use parser options appropriate to the actual format. Treat malformed CSV, encoding errors, and unexpected columns as file or row errors according to the contract. A parser’s permissive recovery is not automatically acceptable data behavior.

## Validate Domain Values

Schema validity is not domain validity. For each row, validate:

* SKU syntax, length, and tenant scope;
* name encoding, length, and prohibited control characters;
* `price_cents` as a non-negative integer, never a floating-point price;
* `active` against an explicit set such as `0` and `1`;
* required fields and whitespace policy;
* duplicate SKUs within the file; existing tenant rows follow the declared upsert policy;
* business limits such as maximum catalog size.

Normalize only by declared rule. Trimming a SKU or lowercasing it can merge two values that the source system considered distinct. If SKU comparison is case-insensitive, enforce that rule with a canonical value and a database constraint. Keep the original value in a bounded error report when operators need to repair the source, and protect that report as tenant data.

## Make the Import a State Machine

An import record should make lifecycle and ownership visible:

~~~text
uploaded → quarantined → scanning → ready → processing
                                      ├─ completed
                                      ├─ completed_with_errors
                                      ├─ failed
                                      └─ cancelled
~~~

Persist status, tenant, actor, source object key, content hash, row limit, parser/policy version, counters, cursor, timestamps, and failure category. Permit only documented transitions. A cleanup task must not delete a file while a processing job owns a valid lease.

The job cursor can be a row number for a stable source object, but row number alone is not enough if the parser or policy changes. Pin the input hash, format version, parser version, and transformation policy to the import. On retry, resume the same interpretation or deliberately restart with a new import ID.

## Choose Transaction and Idempotency Boundaries

Do not wrap the whole file in one transaction. A 500,000-row transaction can hold locks, consume logs, block other work, and make recovery expensive. Process bounded batches, for example 500 rows, in short transactions. Record the cursor and counters only after the batch’s data transaction commits.

The write invariant should be explicit:

~~~text
unique(tenant_id, sku)
one source row maps to one deterministic upsert decision
retrying a committed batch does not duplicate or corrupt rows
cursor advances only after its batch outcome is durable
~~~

Use a stable row identity such as `(import_id, row_number)` for row outcomes. If an import is allowed to update an existing SKU, define last-write and version policy. If concurrent catalog edits are possible, compare a version or acquire an agreed lock; do not let a retry overwrite a newer manual change merely because the source row is old.

An idempotency key for submission prevents duplicate import records after an HTTP timeout. It does not by itself make row writes safe. Batch identity, unique constraints, upsert semantics, and cursor advancement work together.

## Handle Failure and Retry

Classify failures:

| Failure | Action |
| --- | --- |
| malformed row | record row error, continue or stop per contract |
| missing source object | fail import; do not retry forever |
| database deadlock | roll back batch and retry with bounded backoff |
| database constraint conflict | classify as data/concurrency issue |
| worker timeout | release lease; resume from durable cursor |
| storage outage | retry job with backoff and alert |
| notification timeout | reconcile outbox record; do not rerun data batch |
| operator cancellation | stop at a batch boundary and record outcome |

An ambiguous database connection failure requires checking durable batch evidence before replay. If the transaction may have committed, a retry should use the stable row identities and upsert rules, not blindly insert duplicates. Chapter 242 covers idempotency; Chapter 243 covers delivery; Chapter 241 covers unknown completion.

## Report Results Safely

The import report should include total rows seen, valid rows, written rows, skipped rows, error count, first and last processed row, duration, policy version, and final state. Cap row-level errors and aggregate repeated categories so a malicious file cannot create an unbounded report. Store enough evidence for repair without copying sensitive values into logs.

Do not expose another tenant’s import status by guessing IDs. Use opaque IDs, authorization-scoped lookups, and consistent missing/forbidden behavior. An operator may have a broader capability, but it should be explicit and audited.

## Tests That Matter

Test the boundaries in layers:

* upload size, storage key, filename, MIME, and content policy;
* dangerous paths, control characters, malformed encodings, and archive expansion limits;
* exact headers, column count, empty file, blank row, long field, and malformed quote cases;
* integer money parsing, SKU normalization, duplicate rows, and tenant uniqueness;
* deterministic row error ordering and report caps;
* batch commit and cursor advancement after success and failure;
* retry after deadlock, worker timeout, lost response, and ambiguous commit;
* concurrent manual catalog edits and import upserts;
* cancellation, expiration, cleanup, and lease recovery;
* authorization across upload, status, download, cancel, and operator repair;
* notification delivery after import state commits.

Use fixture files large enough to expose memory growth and line-size behavior. Use a real database integration test for unique constraints, batch transactions, cursor recovery, and concurrent edits. Use isolated storage and a malware-scanner/provider fake with explicit accepted, rejected, and unknown outcomes. A unit test over three in-memory rows cannot prove the process or storage limits.

## Observe Resource Use

Track import count and outcome, rows per second, batch duration, memory high-water mark, database lock time, retry count, queue age, source bytes, error categories, report size, and retention cleanup lag. Dimensions should be bounded by tenant class, import type, policy version, and outcome—not raw filenames, SKUs, paths, or arbitrary row values.

Alert on stuck leases, cursor age, repeated batch failures, storage/hash mismatches, memory growth, queue backlog, cleanup failures, and imports exceeding a normal duration. A successful import with a suspiciously small row count can be a semantic failure; compare expected file metadata and business counts where available.

## Rollout and Recovery

Deploy the upload and status API before enabling writes. First run a dry-run validator that stores only a bounded report. Then enable one tenant or import type, compare row outcomes with the existing process, and expand after memory, lock, duration, and error evidence is acceptable.

Keep source files and reports under explicit retention and deletion policies. If a new parser or policy changes interpretation, version it and do not resume an old import under the new rules. If a worker release is rolled back, it must still understand the import state, cursor, batch identity, and source format of records it may claim. Recovery may require forward processing with the old policy, a new repair import, or operator review.

## Common Mistakes

* Trusting a filename, extension, or client MIME type as content validation.
* Writing an uploaded file beneath the document root or using its original path.
* Reading the entire file into memory in a PHP-FPM request.
* Supporting archives without decompressed-size, file-count, and path limits.
* Treating CSV parser recovery as valid business data.
* Parsing prices as floating point values.
* Silently choosing a winner for duplicate rows.
* Advancing the cursor before the batch transaction commits.
* Retrying a lost database response by blindly inserting the batch again.
* Running one transaction for the entire file.
* Letting a cleanup job delete a file held by an active worker.
* Reporting raw filenames, paths, rows, or tenant data in unbounded metrics.
* Sending completion before import state is durable.

## Senior Engineer Thinking

The senior question is not “can PHP read this CSV?” It is “what untrusted bytes are accepted, what interpretation is pinned, which invariants survive partial batches and retries, how is progress proven, and how can an operator recover without replaying uncertain work blindly?”

File import is a pipeline across HTTP, storage, parsing, validation, database, queue, and reporting boundaries. Keep each boundary bounded and observable. Make the source immutable for the run, the cursor durable, the writes idempotent, the errors actionable, and the retention policy explicit.

## Exercises

1. Design the import state machine and list every legal transition, owner, lease, and cleanup rule.
2. Generate a 50 MiB CSV with malformed rows and measure parser memory, row throughput, and report growth.
3. Design a batch write protocol for `unique(tenant_id, sku)` that remains safe after a worker timeout.
4. Threat-model a ZIP upload, including decompression bombs, symlinks, nested archives, and executable content.
5. Define a tenant-scoped import report with error caps, retention, authorization, and operator repair evidence.

## Review Questions

* Why is an upload boundary different from a parser boundary?
* Why should a source object be quarantined before asynchronous processing?
* Which limits must be enforced before reading all file bytes?
* Why is integer cents safer than floating-point price parsing?
* What does a durable cursor prove, and what must accompany it?
* Why are batch identity and database uniqueness both needed for retries?
* How should an ambiguous database commit be handled?
* Which cleanup and retention races can affect an active worker?
* Why can an import complete technically but fail semantically?
* What evidence is needed before changing a parser or policy version?

## Summary

A file importer is an untrusted-input and durable-work pipeline. Define file, row, lifecycle, ownership, retention, and effect contracts; quarantine uploads with server-side keys; bound bytes, rows, fields, archives, and memory; stream and validate deterministically; process durable batches with unique and idempotent write rules; advance cursors only after commit; classify failures and unknown outcomes; report bounded evidence; authorize every management path; observe resource use; and roll out parser changes with pinned versions and recovery plans.

## References

- [PHP Manual: Handling file uploads](https://www.php.net/manual/en/features.file-upload.php)
- [PHP Manual: `fgetcsv()`](https://www.php.net/manual/en/function.fgetcsv.php)
- [Chapter 147 — SSRF](../10-security/147-ssrf.md)
- [Chapter 150 — File Upload Security](../10-security/150-file-upload-security.md)
- [Chapter 120 — Large Datasets](../08-databases/120-large-datasets.md)
- [Chapter 114 — Transactions](../08-databases/114-transactions.md)
- [Chapter 242 — Idempotency](../16-distributed-systems/242-idempotency.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 281 — Rate Limiter](./281-rate-limiter.md)
- [Chapter 282 — URL Shortener](./282-url-shortener.md)
