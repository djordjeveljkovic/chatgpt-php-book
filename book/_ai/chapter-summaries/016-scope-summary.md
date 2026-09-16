# AI Summary — Chapter 16 — Scope

- Status: complete
- Volume: Volume 2 — PHP LANGUAGE FUNDAMENTALS
- Last updated: 2026-09-14

## Written material

Complete chapter covering global/function scope, absent block scope, `global`, `$GLOBALS`, static locals and properties, closures and arrow captures, class scope, namespaces, included-file scope, superglobals, Zend contexts, worker lifetime, database boundaries, security, concurrency, testing, exercises, review questions, and official references.

## Concepts already explained

Name visibility, mutation authority, lifetime, process sharing; explicit dependency passing; static state hazards; closure value/reference capture; include-site scope; namespace versus variable isolation; request-local versus worker-local versus durable state.

## Terminology established

Function scope, global scope, static local, static property, closure capture, reference capture, include-site scope, object context, namespace resolution, process-local state, durable shared state.

## Examples used

Global and function variables; `global` counter; static sequence; value/reference/arrow closures; static property; explicit reservation policy; worker handler; global/static/SQL anti-pattern; repository replacement.

## Cross-references

Builds on Chapters 9, 12, and 15 and prepares object scope, closures/generators, PHP-FPM workers, databases, queues, security, and runtime memory chapters.

## Open threads

Detailed zvals/references, object visibility/inheritance, worker memory, and dependency-injection composition belong in later volumes.

## Exact next section

Chapter 17 — Arrays: the Why This Matters section.

## Technical verification notes

Version-sensitive and scope claims checked against the PHP Manual variable-scope, anonymous-function, arrow-function, and include documentation. The chapter distinguishes process-local PHP state from durable shared state and records arrow capture and include-site behavior.

## Writing notes

Status is complete. Preserve scope as an ownership/lifetime topic, not merely a name-resolution topic.
