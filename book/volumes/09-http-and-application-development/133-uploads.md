---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 133
title: Uploads
slug: uploads
status: complete
summary: ../../_ai/chapter-summaries/133-uploads-summary.md
---

# Chapter 133 — Uploads

## Why This Matters

An uploaded file is attacker-controlled bytes plus metadata supplied by a client and interpreted by a web server, PHP, image libraries, scanners, and storage systems. Checking only the filename extension is not a file-upload policy. A safe upload flow limits resource use, verifies the upload, determines an allowed type, stores it outside executable paths, and records an application-controlled identity.

## Mental Model

```text
request upload
   ↓
transport and size checks
   ↓
temporary file + error validation
   ↓
content/type/policy validation
   ↓
quarantine or durable object storage
   ↓
database metadata and access control
```

Do not publish the temporary name or use the client filename as the storage key. The database record should link an opaque application ID to storage metadata, ownership, policy, and scan state.

## Validate the Upload

PHP exposes upload metadata in `$_FILES`, but every field is untrusted. Check the upload error, size, and that the temporary path is an actual HTTP upload:

```php
<?php

declare(strict_types=1);

function acceptUpload(array $file, string $destination): string
{
    if (($file['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_OK) {
        throw new RuntimeException('Upload failed');
    }
    if (!is_int($file['size'] ?? null) || $file['size'] > 10_000_000) {
        throw new RuntimeException('File is too large');
    }
    if (!is_string($file['tmp_name'] ?? null) || !is_uploaded_file($file['tmp_name'])) {
        throw new RuntimeException('Invalid upload');
    }

    $mime = (new finfo(FILEINFO_MIME_TYPE))->file($file['tmp_name']);
    $allowed = ['image/jpeg' => 'jpg', 'image/png' => 'png', 'application/pdf' => 'pdf'];
    $extension = $allowed[$mime] ?? throw new RuntimeException('Type not allowed');
    $name = bin2hex(random_bytes(16)) . '.' . $extension;
    $path = rtrim($destination, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR . $name;

    if (!move_uploaded_file($file['tmp_name'], $path)) {
        throw new RuntimeException('Could not store upload');
    }

    return $name;
}
```

`finfo` is evidence about the bytes, not a complete malware or business-policy decision. A polyglot file can satisfy more than one parser. For images, decode and re-encode with a trusted library when the policy requires it; for documents, use an antivirus or content-scanning pipeline appropriate to the threat model.

## Storage and Serving

Store uploads outside the web root or in private object storage. Serve them through an authorized application endpoint or a short-lived signed URL. Set a safe `Content-Disposition`, an explicit content type selected by policy, and `X-Content-Type-Options: nosniff` where appropriate. Do not let a client choose an executable extension or storage path.

A download endpoint must authorize the current actor against the database record before reading the object. The object key alone is not authorization. Avoid path traversal by never joining raw client paths; resolve only opaque IDs to server-owned paths.

## Resource Limits and Failure

Set web-server, PHP, request, and application limits consistently: maximum body size, per-file size, file count, memory, processing time, and storage quota. Reject oversized requests before expensive parsing when the infrastructure supports it. Clean temporary files after every failure and monitor disk utilization.

Do not hold a database transaction open while uploading gigabytes or waiting for a scanner. Use a state machine such as `pending`, `scanning`, `available`, `rejected`, and `deleted`; update metadata and scan results transactionally. A worker can retry scanning idempotently by object ID.

## Security and Testing

Threats include path traversal, executable uploads, decompression bombs, oversized images, malicious metadata, SSRF through remote import features, and unauthorized downloads. Restrict parsers and libraries, set timeouts, and isolate scanning where the risk warrants it.

Test upload errors, zero-byte files, size boundaries, misleading extensions, invalid MIME content, duplicate submissions, interrupted storage, scanner failures, unauthorized reads, and deletion races. Use fixture files that exercise the actual decoder and storage adapter rather than testing only arrays of metadata.

## Common Mistakes

- Trusting `name`, extension, or client MIME type.
- Storing files under a client-controlled path.
- Serving private files directly from a public directory.
- Processing unbounded archives or images synchronously.
- Forgetting cleanup after a failed move or scan.
- Recording metadata as available before the object is durable and approved.

## Senior Engineer Thinking

File handling crosses several trust boundaries. Define the allowed content, maximum resources, storage lifecycle, scanning decision, and download authorization. Keep the uploaded bytes separate from user-controlled presentation and use an opaque identity so renaming, migration, and revocation remain possible.

## Exercises

1. Extend the example with a database metadata record and a quarantine state.
2. Design limits for a profile-image endpoint and a document-import endpoint.
3. Write tests for a file with a valid extension but disallowed detected MIME type.
4. Design an authorized download response for a private object store.

## Review Questions

1. Why is the client filename not a safe storage key?
2. What does `is_uploaded_file()` prove and what does it not prove?
3. Why should private files be served through an authorization boundary?
4. Which upload work belongs in a queue rather than in the request?

## Summary

Treat uploads as hostile bytes and metadata. Validate transport, size, content, storage state, and policy; generate server-owned names; keep objects private by default; scan or transform risky content; and authorize every download. Bound resources, clean failures, and test the real storage and parser paths.

## References

- [OWASP: File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [PHP Manual: Handling file uploads](https://www.php.net/manual/en/features.file-upload.php)
- [PHP Manual: `finfo_file`](https://www.php.net/manual/en/function.finfo-file.php)
