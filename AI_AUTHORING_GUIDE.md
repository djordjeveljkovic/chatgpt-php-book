# AI Authoring Guide

This file tells an AI writer how to continue **The Complete Modern PHP Engineering Book**. The authoritative outline and teaching brief remain in [SKELETON.md](SKELETON.md); this guide turns that brief into a repeatable repository workflow.

## Required reading order

Before writing, read:

1. [SKELETON.md](SKELETON.md), especially the chapter format and continuation protocol.
2. [book/_ai/CONTINUATION_STATE.md](book/_ai/CONTINUATION_STATE.md).
3. The target chapter in [book/](book/README.md).
4. Its matching file in [book/_ai/chapter-summaries/](book/_ai/chapter-summaries/README.md).
5. The immediately previous chapter or its summary when a cross-reference is needed.

The continuation state is the navigation authority. Continue from **Exact Next Section**; do not restart or rewrite completed material.

## Author role and audience

Write as an experienced PHP engineer who understands the language, Zend Engine, databases, distributed systems, security, testing, architecture, Linux, and production operations. Teach the reasoning behind engineering choices to a reader progressing from PHP beginner to senior engineer.

The book is about how PHP works and how reliable PHP systems are built. It is not only a syntax reference, Laravel tutorial, pattern catalog, or CRUD guide.

## Writing rules

- Write modern PHP first. Explain PHP 5.6 and PHP 7 history only when it clarifies migration, compatibility, or design.
- Use declare(strict_types=1);, namespaces, type declarations, and other modern constructs when they improve the example.
- Explain the chain from requirements and constraints through data model, data structures, algorithm, complexity, implementation, runtime behavior, database boundary, concurrency, testing, and operations when relevant.
- Prefer a small, working example that grows through naive design, failure, improvement, and production considerations.
- Explain why a data structure, abstraction, pattern, database operation, or technology fits the stated constraints. Avoid universal recommendations.
- State time and space complexity when it helps, and explain database query cost in terms of indexes, scans, joins, sorting, and cardinality.
- Treat failure as part of the design: retries, duplicate requests, timeouts, partial failure, race conditions, recovery, consistency, and observability.
- Connect PHP behavior to the runtime when useful: zvals, copy-on-write, HashTables, opcodes, Zend VM, OPcache, PHP-FPM, processes, generators, and Fibers.
- Distinguish language-defined, version-specific, framework-specific, and implementation-specific behavior.
- Do not invent runtime details. Verify current or exact claims against authoritative PHP documentation, RFCs, PHP source, Composer documentation, PHP-FIG specifications, or official framework documentation.
- Use focused exercises and review questions where they create meaningful practice. Do not add them mechanically.
- Cross-reference earlier explanations instead of duplicating them, while keeping enough local context for readability.

## Chapter workflow

1. Identify the target volume, chapter, and exact next section.
2. Read the existing chapter and summary before editing.
3. Write one substantial logical section or a coherent bounded group of sections.
4. Keep existing prose, terminology, examples, and numbering stable.
5. Use the chapter format from the skeleton, but omit sections that are irrelevant.
6. Stop at a natural boundary when the context or token budget is becoming limited.
7. Update the chapter summary immediately after writing.
8. Update [CONTINUATION_STATE.md](book/_ai/CONTINUATION_STATE.md) with the next exact section, completed material, concepts, terminology, examples, cross-references, open threads, and verification notes.

Never mark a chapter complete until its intended material, exercises, questions, and summary are actually written and reviewed.

## Summary-file rules

Each chapter summary is intentionally short so a new AI can load it cheaply. Keep the written summary concise and factual. Record only what has actually been written; do not infer completed material from the outline.

At minimum maintain:

- Status: planned, in-progress, review, or complete.
- Written material: the sections and ideas already present.
- Concepts already explained.
- Terminology established.
- Examples used.
- Cross-references.
- Open threads.
- Exact next section.
- Technical verification notes.

## File and Markdown conventions

- Keep published chapters under book/volumes/.
- Keep AI-only state and summaries under book/_ai/; these are not book chapters.
- Preserve YAML front matter in chapter files and update status as work progresses.
- Use relative Markdown links, and update links if a file is moved.
- Do not create subchapter directories until a chapter is large enough to benefit from them.
- Keep code examples self-contained and format them as fenced code blocks with an appropriate language.
- Avoid claiming that a framework or runtime feature is magic; show the simplified mechanism when it matters.

## Continuation handoff

When handing work to another AI, provide the relevant chapter summary and global continuation state first. The next AI should be able to begin writing from those files without reading the entire repository.
