# AI Summary — Chapter 18 — Strings

- Status: complete
- Volume: Volume 2 — PHP LANGUAGE FUNDAMENTALS
- Last updated: 2026-09-14

## Written material

The complete chapter explains PHP strings as byte sequences and separates binary data from Unicode text. It covers literal forms, concatenation, byte versus character length, UTF-8 and grapheme concerns, search and comparison, formatting, parsing, stream framing, output contexts, edge cases, complexity, security, database encoding/collation, concurrency boundaries, testing, common mistakes, senior-engineer reasoning, exercises, and review questions.

## Concepts already explained

- Byte strings, encoding contracts, Unicode code points, grapheme clusters
- Heredoc/nowdoc, interpolation, strict position checks, constant-time secret comparison
- Parsing and framing, size limits, context-specific escaping and parameterization
- Canonicalization, malformed input, stream processing, database collation

## Terminology established

Byte sequence, text encoding, UTF-8, code point, grapheme cluster, binary data, output sink, canonicalization, framing, byte limit.

## Examples used

UTF-8 display-name validation; context-specific HTML and SQL handling; a bounded line-protocol reader; unsafe and corrected HTML rendering; string boundary tests.

## Cross-references

Connects to the request/response boundary and later HTTP, security, database, and performance volumes.

## Open threads

Later material can cover complete HTTP encoding negotiation, internationalization policies, regular-expression internals, and search indexing.

## Exact next section

Chapter complete; no next section.

## Technical verification notes

The chapter treats PHP's core strings as bytes and assigns Unicode behavior to mbstring/Intl-style libraries. It avoids claiming that byte and character operations have identical contracts or that escaping is interchangeable across sinks.
