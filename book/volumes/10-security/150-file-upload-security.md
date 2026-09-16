---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 150
title: File Upload Security
slug: file-upload-security
status: complete
summary: ../../_ai/chapter-summaries/150-file-upload-security-summary.md
---

# Chapter 150 — File Upload Security

## Why This Matters

Uploads combine attacker-controlled bytes, metadata, parsers, storage, and later downloads. A valid image extension does not prove that a file is safe to decode; a successfully moved temporary file does not prove that its contents are approved. Upload security needs layered limits and a lifecycle that can quarantine, scan, reject, and revoke an object.

## Layered Policy

Define the policy before writing the controller:

```text
actor and quota
 → request/file size
 → upload error and transport origin
 → content type and parser validation
 → malware/scanning policy
 → private storage and metadata
 → authorized download
```

Each layer answers a different question. Do not collapse them into one extension check.

## Content and Resource Checks

Use server limits, `is_uploaded_file()`, `finfo`, and a trusted decoder where appropriate. For images, decode and re-encode to remove active metadata or malformed structures if the product permits. For archives, limit nesting, expanded size, entry count, and symlink behavior before extraction. Never extract an archive directly into the web root.

Keep uploads outside executable paths and generate random object keys. Store the original filename only as untrusted display metadata. Keep it separate from the `Content-Disposition` value and encode it for that header's grammar.

## Quarantine State

Record an object as `pending` or `quarantined` until validation and scanning finish. A worker can transition it to `available` or `rejected`; downloads authorize both actor and state. Make transitions idempotent by object ID and retain a reason code that does not expose scanner internals to the user.

```php
<?php

declare(strict_types=1);

enum UploadState: string
{
    case Quarantined = 'quarantined';
    case Available = 'available';
    case Rejected = 'rejected';
}

function canDownload(UploadState $state, bool $authorized): bool
{
    return $authorized && $state === UploadState::Available;
}
```

The enum is application policy, not a replacement for storage ACLs or a scanner. Keep the object store private so a leaked key cannot bypass the state machine.

## Failure and Operations

Uploads consume disk, CPU, memory, temporary files, queue capacity, and scanner capacity. Enforce per-user and global quotas, monitor storage utilization, and clean abandoned quarantine objects. Use request and parser timeouts. Rate limit expensive transforms and scanning operations.

If object storage succeeds but database metadata fails, mark or reconcile the orphan through a repair job. If metadata commits but the object is missing, serve a controlled failure and alert. These are distributed steps; do not pretend a database transaction includes an object-store write.

## Testing and Common Mistakes

Test polyglot and mislabeled files, decompression bombs, oversized images, archive traversal, duplicate and replayed uploads, scanner timeouts, orphan repair, unauthorized downloads, state transitions, and deletion races. Use realistic fixtures and the actual object-store adapter in integration tests.

Common mistakes include trusting client MIME values, allowing active content, scanning synchronously without limits, returning private storage URLs permanently, and deleting metadata before a consumer has stopped using the object.

## Senior Engineer Thinking

Upload security is a lifecycle and resource-control problem. Specify the accepted content, ownership, states, quotas, parsers, scanning decision, storage permissions, download headers, retention, and recovery path. Layered controls remain useful when one parser or scanner is wrong.

## Exercises

1. Design a quarantine schema and idempotent scan worker.
2. Set limits for images, archives, and documents and explain the resource each protects.
3. Write an orphan-reconciliation process for object store and database disagreement.

## Review Questions

1. Why does MIME detection not prove malware safety?
2. What should a quarantine state prevent?
3. Which failures can leave orphan objects?
4. Why must private object storage enforce access independently of application URLs?

## Summary

Secure uploads with layered size, content, parser, scanning, storage, quota, and authorization controls. Use random keys, private storage, quarantine states, bounded processing, idempotent workers, and repair paths for partial failures.

## References

- [OWASP: File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [PHP Manual: File uploads](https://www.php.net/manual/en/features.file-upload.php)
- [CWE-434: Unrestricted Upload of File with Dangerous Type](https://cwe.mitre.org/data/definitions/434.html)
