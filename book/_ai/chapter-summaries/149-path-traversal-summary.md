# AI Summary — Chapter 149 — Path Traversal

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains traversal through file paths, opaque object IDs, canonicalization, symlinks and race limits, platform-specific paths, stream wrappers, private serving, testing, exercises, and review questions.

## Concepts already explained

- Client-selected paths are unsafe; server-owned storage keys and authorized records are stronger boundaries.
- Canonicalization supports containment checks but does not by itself solve symlink races or authorization.

## Terminology established

Path traversal, canonical path, containment check, symlink race, stream wrapper, opaque object ID.

## Examples used

- Vulnerable concatenated download and `realpath()` containment helper.
- Cross-platform path, private storage, and authorized download policies.

## Cross-references

- [Chapter 133 — Uploads](../../volumes/09-http-and-application-development/133-uploads.md)
- [Chapter 150 — File Upload Security](../../volumes/10-security/150-file-upload-security.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 150 — File Upload Security: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated security proofread. Path guidance links to OWASP and PHP filesystem/stream documentation.
