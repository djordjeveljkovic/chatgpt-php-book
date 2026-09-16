---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 68
title: Filesystem
slug: filesystem
status: complete
summary: ../../_ai/chapter-summaries/068-filesystem-summary.md
---

# Chapter 68 — Filesystem

## Why This Matters

The filesystem is shared mutable state with permissions, latency, caching, crashes, and races. A file may be incomplete while another request reads it. A `file_exists()` check may be true while the following `fopen()` fails. A deployment that overwrites a file in place can expose a partial document. A path assembled from user input can escape its intended directory.

Reliable filesystem code treats paths and contents as separate concerns. Validate the name, resolve the allowed root, choose a write protocol, check every operation, and define crash recovery.

## Mental Model

```text
logical object ID → approved directory + generated filename
                  → temporary complete write
                  → close successfully
                  → atomic rename/publication
                  → readers see old or new version by policy
                  → cleanup and reconciliation
```

A filesystem call crosses into the operating system. PHP’s return value tells you what PHP observed; it does not by itself promise survival after power loss.

## Core Concept: Paths Are Data

A path is not safe merely because it has no visible `../`. Encodings, separators, symlinks, null bytes, races, and alternate wrappers complicate validation. Prefer generating final filenames from internal identifiers and storing user-supplied names as display metadata.

```text
client filename: "holiday.jpg"  → never used as authority
internal ID:     01J...         → /srv/uploads/ab/cd/<id>.bin
```

Inspect content using trusted rules rather than only an extension or client MIME header.

## Minimal Example: Safe Directory Creation

```php
<?php

declare(strict_types=1);

function ensureDirectory(string $directory): void
{
    if (is_dir($directory)) {
        return;
    }
    if (! mkdir($directory, 0750, true) && ! is_dir($directory)) {
        throw new RuntimeException("Unable to create directory: {$directory}");
    }
}
```

The second check handles the race in which another process creates the directory between the check and `mkdir()`. The final mode is also affected by the process umask.

## Atomic Publication

Do not truncate a path that concurrent readers use. Write a temporary file in the destination directory, close it successfully, then rename it:

```php
<?php

declare(strict_types=1);

function atomicWrite(string $destination, string $contents): void
{
    $directory = dirname($destination);
    ensureDirectory($directory);
    $temporary = tempnam($directory, '.write-');
    if ($temporary === false) {
        throw new RuntimeException('Unable to allocate temporary file.');
    }

    try {
        $bytes = file_put_contents($temporary, $contents, LOCK_EX);
        if ($bytes === false || $bytes !== strlen($contents)) {
            throw new RuntimeException('Temporary write was incomplete.');
        }
        if (! rename($temporary, $destination)) {
            throw new RuntimeException('Unable to publish temporary file.');
        }
        $temporary = null;
    } finally {
        if (is_string($temporary) && is_file($temporary)) {
            unlink($temporary);
        }
    }
}
```

Same-directory replacement matters because cross-filesystem rename behavior differs. Rename generally gives readers an old-or-new directory entry within one filesystem; durability after power loss and visibility to other hosts require deployment-specific guarantees.

## Locking and Races

`flock()` coordinates cooperating processes using the same protocol. It does not lock every program that opens the file and does not make a check-then-use sequence atomic. For “this name must not exist,” an exclusive creation mode can express the invariant more directly.

```php
$handle = fopen($lockPath, 'c');
if ($handle === false || ! flock($handle, LOCK_EX | LOCK_NB)) {
    throw new RuntimeException('Another process owns the lock.');
}
try {
    // Re-read shared state after acquiring the lock.
} finally {
    flock($handle, LOCK_UN);
    fclose($handle);
}
```

The re-read matters because state can change while waiting. A marker file can remain after a crash; an advisory lock held by a dead process is normally released by the OS. Choose the protocol intentionally.

## Bad Example and Better Design

```php
$path = __DIR__ . '/uploads/' . $_POST['filename'];
move_uploaded_file($_FILES['file']['tmp_name'], $path);
```

The client controls the path, names collide, permissions and content policy are unclear, and the return value is ignored. Instead generate an object ID, map it to a controlled path, stream to a temporary file with a byte limit, inspect the content, and publish only after validation. Keep uploaded bytes outside an executable document root and serve them through an authorization-aware path.

## Lifecycle and Failure Modes

```text
write starts
 ├─ crash before close → temporary orphan; janitor removes it
 ├─ rename fails       → old destination remains; report failure
 ├─ reader during swap → old or new entry by filesystem rules
 └─ disk full/quota    → retry may not help; alert and reject safely
```

Do not use `file_exists()` as a reservation. Attempt the operation and handle its result. `realpath()` canonicalizes existing paths but returns `false` for missing paths and does not remove race conditions; authorization must not depend on a stale check.

## Performance

Filesystem cost depends on storage, cache state, file count, directory layout, size, and concurrency. Stream large files instead of reading them into memory. Shard large object collections when lookup and operations require it. Recursive scans and metadata calls can be expensive on network filesystems. Measure p95/p99 latency and errors, not just local throughput.

## Security

Use least-privilege directories, avoid executable upload locations, reject traversal, constrain symlink behavior, and never trust an extension. Protect backups and temporary files. Cleanup jobs must resolve exact targets and reject empty or broad roots. File permissions are not a business authorization model; enforce ownership in application or database state.

## Database Interaction

The filesystem is often suitable for large immutable blobs, while the database stores identity, owner, checksum, size, media type, and publication state. Use a transaction for metadata and an outbox or reconciliation job for the blob boundary:

```text
write temporary blob → publish blob → commit available metadata
       └────────────── crash anywhere ──────────────┘
                   reconciliation repairs orphan/missing state
```

If metadata commits before the blob, readers need a pending state. If the blob publishes first, a crash can leave an orphan. Neither ordering eliminates recovery work; each makes it explicit.

## Testing

Test traversal, duplicate IDs, missing directories, simulated write failures, interrupted writes, stale temporary files, concurrent publication, permissions, and cleanup boundaries. Use a temporary directory per test. Integration-test the actual filesystem type when rename, ownership, or symlink semantics matter.

## Common Mistakes

- Building paths directly from user filenames.
- Treating `file_exists()` followed by an operation as atomic.
- Writing directly to a path read by consumers.
- Assuming `LOCK_EX` solves multi-host consistency.
- Ignoring partial writes and cleanup failures.
- Serving uploaded content from an executable document root.

## Senior Engineer Thinking

Define the namespace, object owner, publication point, reader view during replacement, and reconciliation after a crash. Once files participate in a distributed workflow, “save this string” is no longer the complete requirement.

## Exercises

1. Implement an atomic JSON configuration writer and test concurrent readers.
2. Build a checksum-based object path and handle duplicate uploads safely.
3. Design cleanup for abandoned temporary files without deleting active uploads.
4. Compare a filesystem lock with a database lock for a multi-host deployment.

## Review Questions

1. Why should the temporary file be in the destination directory?
2. What does `flock()` coordinate and not coordinate?
3. Why is `file_exists()` not a reservation?
4. Which metadata belongs in the database when blobs live on disk?
5. What recovery exists after a crash before or after publication?

## Summary

Filesystem work is concurrent, failure-prone shared state. Separate identifiers from paths, use least privilege, stream bounded data, publish complete files atomically where appropriate, coordinate with explicit protocols, and reconcile file/database boundaries. The next chapter moves from files and pipes to creating and supervising processes.

## References

- [PHP Manual: Filesystem Functions](https://www.php.net/manual/en/ref.filesystem.php)
- [PHP Manual: `rename()`](https://www.php.net/manual/en/function.rename.php)
- [PHP Manual: `flock()`](https://www.php.net/manual/en/function.flock.php)
- [PHP Manual: `tempnam()`](https://www.php.net/manual/en/function.tempnam.php)
- [PHP Manual: `realpath()`](https://www.php.net/manual/en/function.realpath.php)
- [PHP Manual: Handling file uploads](https://www.php.net/manual/en/features.file-upload.php)
