# AI Summary — Chapter 96 — Semantic Versioning

- Status: complete
- Volume: Volume 7 — COMPOSER AND THE PHP ECOSYSTEM
- Last updated: 2026-09-15

## Written material

The chapter introduces SemVer 2.0.0 as a publisher's compatibility contract, explains major/minor/patch releases, defining the public API, immutable releases, the weaker expectations around `0.x`, and the real-world limits of version promises. It distinguishes pre-release precedence from Composer stability filtering, and explains that build metadata does not affect SemVer precedence.

It then distinguishes strict SemVer from Composer behavior: VCS `v` tag prefixes, internal numeric normalization, development branches, caret/tilde/wildcard/comparison/union/hyphen constraints, exact pins, stability flags, and the difference between a manifest range and the lock-file selection. It closes with author/consumer policy, compatibility testing, operational guidance, mistakes, exercises, and review questions.

## Concepts already explained

- After `1.0.0`, patches are compatible fixes, minors add compatible API or deprecations, and majors indicate breaking public API changes.
- Public API can include interfaces, named arguments, configuration, exceptions, serialized formats, and observable defaults—not only class and function names.
- SemVer `0.y.z` releases do not promise a stable API; Composer's conservative caret behavior is a range rule, not a package guarantee.
- SemVer prerelease identifiers affect precedence; build metadata does not. Composer stability policy separately determines candidate eligibility.
- Composer constraints are Composer syntax, not part of SemVer. The manifest constraint expresses allowed versions; `composer.lock` captures the selected installation for an application.
- A successful resolution and lowest/highest dependency jobs provide evidence, not proof of full behavioral compatibility.

## Terminology established

SemVer, public API, compatibility promise, pre-release, build metadata, Composer constraint, normalized version, development branch, stability flag, `minimum-stability`, `prefer-stable`, manifest range, lock-file selection.

## Examples used

- `^1.4.2`, `~1.4.2`, `~1.4`, `1.4.*`, `>=1.4 <2.0`, `^1.4 || ^2.0`, and partial hyphen range `1.2 - 2.0`.
- `^0.4.2` and `^0.0.5` to show Composer's pre-1.0 caret bounds.
- `2.1.0-alpha.2`, `2.1.0-alpha.10`, `2.1.0-rc.1`, and `2.1.0+build.42` for precedence and metadata.
- A root `require` example for `monolog/monolog`, and a package-specific `@beta` stability example.
- `composer update --prefer-lowest --prefer-stable` as a lower-bound compatibility job.

## Cross-references

- [Chapter 93 — Dependency Resolution](../../volumes/07-composer-and-the-php-ecosystem/093-dependency-resolution.md)
- [Chapter 94 — composer.json](../../volumes/07-composer-and-the-php-ecosystem/094-composer-json.md)
- [Chapter 95 — composer.lock](../../volumes/07-composer-and-the-php-ecosystem/095-composer-lock.md)
- The volume outline is in [SKELETON.md](../../../SKELETON.md).

## Open threads

No open chapter work remains.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

Checked SemVer 2.0.0 rules and Composer constraint/tag/stability behavior against the official SemVer specification and Composer documentation. Composer's `composer/semver` README and `VersionParser` source are cited for normalization/parser boundaries. Ran 19 constraint assertions through the Composer 2.10.3 bundled `composer/semver` library, parsed both JSON examples, checked all local links in the chapter and summary, and ran `git diff --check`; all passed. Independent proofreading found no blocking correctness issues; wording around tilde ranges, parser compatibility, and equal precedence for build-metadata variants was clarified.

## Writing notes

Keep lock-file mechanics in Chapter 95 and dependency candidate/solver behavior in Chapter 93; this chapter covers compatibility semantics and constraint interpretation.
