---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 149
title: Path Traversal
slug: path-traversal
status: complete
summary: ../../_ai/chapter-summaries/149-path-traversal-summary.md
---

# Chapter 149 — Path Traversal

## Why This Matters

Path traversal occurs when input intended to name a file is interpreted as a filesystem path. `../`, encoded separators, alternate stream syntax, symlinks, and platform-specific rules can escape an intended directory and expose or overwrite files. A filename is not an authorization decision.

## Mental Model

```text
client identifier → authorized record → server-owned path
```

Prefer opaque IDs mapped to a database record and a storage key. If a user must supply a relative name, validate it, canonicalize it, and verify that the resolved path remains inside an approved directory. The check and the file operation must account for symlink and race behavior.

## Vulnerable and Safer Code

This is unsafe:

```php
$path = __DIR__ . '/downloads/' . $_GET['name'];
readfile($path);
```

A safer boundary resolves a server-owned ID:

```php
<?php

declare(strict_types=1);

function privatePath(string $root, string $storedKey): string
{
    $root = realpath($root);
    if ($root === false) {
        throw new RuntimeException('Storage unavailable');
    }

    $candidate = realpath($root . DIRECTORY_SEPARATOR . $storedKey);
    if ($candidate === false || !is_file($candidate)) {
        throw new RuntimeException('Object unavailable');
    }

    $prefix = rtrim($root, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR;
    if (!str_starts_with($candidate, $prefix)) {
        throw new RuntimeException('Invalid storage key');
    }

    return $candidate;
}
```

The database lookup and authorization must happen before this function. `realpath()` requires an existing path and does not by itself eliminate time-of-check/time-of-use races or symlink changes. For writes, create files with exclusive semantics in a storage system that controls keys rather than validating a path and then trusting it.

## Encoding and Platform Details

Decode URL and form input exactly once at the HTTP boundary. Reject NUL bytes and unexpected separators, normalize Unicode policy, and account for Windows drive letters, UNC paths, backslashes, and stream wrappers if the application runs on those platforms. Do not attempt to maintain a hand-written blacklist of every traversal spelling.

Do not allow a client to select `php://`, `data://`, or other stream-wrapper targets. Keep user content outside executable directories and disable script execution in upload and download storage. A path policy must match the PHP configuration and web-server configuration actually deployed.

## Authorization and Headers

The endpoint should authorize the actor against the object record, then stream the file with a server-selected content type and safe disposition. Do not reveal whether an object exists to an unauthorized caller unless the API policy permits it. Avoid using raw names in redirects or `Content-Disposition` without safe encoding.

## Testing and Operations

Test `../`, encoded traversal, repeated separators, backslashes, absolute paths, symlinks, missing parents, Unicode normalization, long names, and IDs belonging to another tenant. Test both reads and writes. Monitor rejected path attempts, storage errors, and unexpected file types without logging sensitive path contents.

## Common Mistakes

- Concatenating a request parameter into a path.
- Checking only for the substring `..`.
- Treating `basename()` as authorization.
- Validating a path but then following a mutable symlink.
- Allowing stream wrappers or user-selected absolute paths.
- Putting private files under a publicly executable directory.

## Senior Engineer Thinking

The strongest path-traversal defense is to remove path choice from the client contract. Store opaque keys, resolve them through authorized metadata, use a storage API with safe key semantics, and isolate private objects. Canonicalization is a supporting check, not the entire policy.

## Exercises

1. Replace a filename-based download endpoint with an opaque object ID.
2. Add tests for encoded traversal and symlink escapes.
3. Define storage and web-server rules that prevent uploaded PHP files from executing.

## Review Questions

1. Why is `basename()` insufficient?
2. What does canonicalization prove, and what race can remain?
3. Which platform-specific path forms must a cross-platform service consider?
4. Why should private storage be outside the executable web root?

## Summary

Prevent path traversal by avoiding client-selected paths, authorizing opaque object IDs, using server-owned storage keys, canonicalizing where necessary, and enforcing private non-executable storage. Test encoded, platform-specific, symlink, and cross-tenant cases.

## References

- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [PHP Manual: Filesystem functions](https://www.php.net/manual/en/book.filesystem.php)
- [PHP Manual: Stream wrappers](https://www.php.net/manual/en/wrappers.php)
