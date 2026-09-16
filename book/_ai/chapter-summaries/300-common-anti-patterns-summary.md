# AI Summary — Chapter 300 — Common Anti-Patterns

- Status: complete
- Volume: Volume 21 — REFERENCE
- Last updated: 2026-09-17

## Written material

Chapter 300 distinguishes context-dependent anti-patterns from style preferences and catalogs service splitting without ownership, interface proliferation, god services, anemic models, ORM pass-through, cache authority, global state, framework magic, retry amplification, unbounded fan-out, synchronous external work, mock-only testing, rewrites, and permanent flags. It includes evaluation questions, a decision table, exercises, and a Chapter 301 handoff.

## Concepts already explained

Context; ownership; substitution; source of truth; retry budget; bounded work; reversibility; migration seam; operational cost; evidence.

## Terminology established

Service decomposition, cache, retries, fan-out, provider calls, legacy rewrite, feature flags, FPM workers.

## Examples used

None.

## Cross-references

[Chapter 223 — Performance Mental Model](../../volumes/15-performance/223-performance-mental-model.md); [Chapter 290 — Technical Debt](../../volumes/20-senior-engineering/290-technical-debt.md); [Chapter 296 — Engineering Trade-Offs](../../volumes/20-senior-engineering/296-engineering-trade-offs.md); [Chapter 301 — Data Structure Guide](../../volumes/21-reference/301-data-structure-guide.md).

## Open threads

Continue with Chapter 301 — Data Structure Guide.

## Exact next section

Chapter 301 — Data Structure Guide: the Why This Matters section.

## Technical verification notes

The source contains no executable PHP blocks. Local Markdown links resolved and `git diff --check` passed. The chapter received a local editorial check for context, ownership, source-of-truth, retries, bounded work, testing, rewrites, and the Chapter 301 handoff. Live architecture, provider, queue, and deployment integrations were not run.

## Writing notes

Keep this summary short and update it after every writing session.
