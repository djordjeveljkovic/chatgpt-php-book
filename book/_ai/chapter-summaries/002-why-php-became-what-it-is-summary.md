# AI Summary — Chapter 2 — Why PHP Became What It Is

- Status: complete
- Volume: Volume 1 — THE PHP MENTAL MODEL
- Last updated: 2026-09-14

## Written material

The chapter explains PHP's evolution as a response to changing engineering constraints: early web-oriented scripting, public adoption, PHP 3 extensibility, PHP 4's Zend Engine rewrite, PHP 5's stronger object model, the abandoned PHP 6 Unicode effort, PHP 7's phpng/Zend Engine 3 runtime redesign, and PHP 8's more explicit language features. It connects this history to compatibility, process models, performance, security, testing, and migration decisions.

## Concepts already explained

- Evolutionary language and runtime design
- Web-centered origins and low-ceremony development
- Extensibility as an ecosystem strategy
- Runtime rewrites and compatibility pressure
- Zend Engine generations as execution foundations
- Gradual movement toward explicit types and contracts
- Historical compatibility versus modern design
- Performance attribution across PHP, databases, networks, and processes
- Boundary validation, migration testing, and characterization tests

## Terminology established

PHP Tools, FI, PHP/FI, PHP 3, PHP 4, Zend Engine, Zend Engine 2, Zend Engine 3, PHP 5, PHP 6 Unicode effort, phpng, PHP 7, PHP 8, extensibility, compatibility, migration, boundary, invariant, JIT compiler, characterization test.

## Examples used

- A historical pressure model from small web scripts to modern runtime and language features.
- A typed readonly `Registration` value object with constructor property promotion.
- An unvalidated array-based queue boundary and a validated conversion into `Registration`.
- A request performance path separating PHP, database, cache, network, serialization, and response costs.

## Cross-references

- Chapter 1 for the language/runtime/application-environment distinction.
- Chapter 3 for a more precise separation of PHP as a language and PHP as a runtime.
- Later runtime chapters for opcodes, zvals, HashTables, function calls, memory management, and OPcache.
- Later legacy, testing, security, and performance volumes for migration and operational treatment.

## Open threads

No chapter-specific open threads remain. Later chapters should deepen the runtime and process-model mechanisms introduced here without repeating the historical timeline.

## Exact next section

None — chapter complete. Continue with Chapter 3 — PHP as a Language vs PHP as a Runtime.

## Technical verification notes

Historical milestones and PHP 8 feature claims were checked against the official PHP Manual history pages, the PHP 8.0 migration guide, and the official PHP 8.0 release announcement. Exact current-version claims were avoided. Performance language is qualified by workload and measurement rather than presented as a universal version guarantee.

## Writing notes

The chapter uses history to explain engineering trade-offs, not to recommend obsolete practices. Modern examples use strict types, a readonly value object, constructor property promotion, and explicit input validation.
