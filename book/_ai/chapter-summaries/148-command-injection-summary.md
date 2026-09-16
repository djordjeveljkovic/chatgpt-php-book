# AI Summary — Chapter 148 — Command Injection

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains shell/process injection, avoiding shells, fixed argument arrays, allow-lists, process privileges and limits, testing, observability, exercises, and review questions.

## Concepts already explained

- Shell commands are programs; concatenated input can alter syntax.
- Libraries or fixed process argument arrays are safer than shell strings, but path authorization, least privilege, and resource bounds remain required.

## Terminology established

Command injection, process boundary, bypass-shell execution, executable allow-list, process isolation, output limit.

## Examples used

- Vulnerable `shell_exec()` concatenation and a controlled `proc_open()` example.
- Fixed operation maps, timeouts, restricted environment, and monitoring.

## Cross-references

- [Chapter 149 — Path Traversal](../../volumes/10-security/149-path-traversal.md)
- [Chapter 150 — File Upload Security](../../volumes/10-security/150-file-upload-security.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 149 — Path Traversal: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated security proofread. Command-execution guidance links to OWASP and the PHP Manual.
