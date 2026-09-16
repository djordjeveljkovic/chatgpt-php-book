---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 96
title: Semantic Versioning
slug: semantic-versioning
status: complete
summary: ../../_ai/chapter-summaries/096-semantic-versioning-summary.md
---

# Chapter 96 — Semantic Versioning

## Why This Matters

A version number is a message from a package author to its users. If `acme/slugger` releases `1.5.0` after `1.4.2`, can a project using `^1.4` take that update safely? If the author publishes `2.0.0`, should the consumer expect to edit its code? Semantic Versioning (SemVer) is a convention for answering those questions through a public compatibility promise.

That promise influences Composer constraints, library release practice, upgrade planning, and CI coverage. It is not proof that an update is safe. Composer can compare version strings and apply constraints, but it cannot inspect all the ways application code depends on a package or force a maintainer to follow SemVer correctly.

## The SemVer Contract

SemVer 2.0.0 uses the form `MAJOR.MINOR.PATCH`, for example `3.7.2`. Its rules apply to software that has declared a precise public API. For a package already at `1.0.0` or later, the intended signals are:

| Change | Version part | Compatibility promise |
| --- | --- | --- |
| Backward-compatible bug fix | Patch | Existing public API remains compatible |
| Backward-compatible public feature or deprecation notice | Minor | Existing public API remains compatible; new API may be added |
| Backward-incompatible public API change | Major | Consumers may need code or configuration changes |

If `1.4.2` is the current release, a compatible bug fix could be `1.4.3`; a backward-compatible feature could be `1.5.0`; removing or changing an existing public contract calls for `2.0.0`. Incrementing the minor version resets patch to zero; incrementing the major resets both minor and patch to zero. The package should not replace the contents of a released version or move a release tag to different code. A changed release gets a new version.

The package author must first decide what the public API is. In a PHP library that can include public classes and functions, method signatures and return types, documented exceptions, default behavior, configuration keys, serialized formats, and command-line behavior. An API can also include names that PHP users rely on through named arguments. A declaration that an API is backward compatible is meaningful only when the supported contract is clear.

“Backward compatible” is judged from the consumer's perspective. Adding an optional argument can be compatible for callers, but adding a required method to an interface can break every implementation maintained by consumers. Changing an exception type, output format, or default may break users even if no class or function was removed. Tests should exercise the published contract, not just implementation internals.

## Version Zero and Real-World Limits

SemVer reserves `0.y.z` for initial development. The specification says the public API should not be considered stable and that anything may change. Therefore `0.4.8` does not carry the same compatibility promise as a `1.4.8` release. Composer's caret operator treats pre-1.0 ranges cautiously, but that is a Composer constraint rule, not a guarantee that a `0.x` package follows SemVer.

Composer interprets `^0.4.2` as accepting versions at least `0.4.2` and below `0.5.0`; it does not cross the next minor line. `^0.0.5` stops before `0.0.6`. These ranges reduce the allowed update surface, but a maintainer can still introduce a breaking change inside `0.4.x`. Review the package's policy and tests before relying on its compatibility claims.

After `1.0.0`, SemVer gives consumers a stronger expectation, not a guarantee enforced by tooling. Package authors sometimes break behavior in a minor release, interpret “public” differently, or depend on upstream packages with incompatible release practices. A test suite and integration review remain necessary.

## Pre-Releases and Build Metadata

A pre-release appends a hyphen and identifiers to the patch version: `2.1.0-alpha.1`, `2.1.0-beta.2`, or `2.1.0-rc.1`. A pre-release has lower precedence than the corresponding final release, so `2.1.0-rc.1 < 2.1.0`. Pre-release identifiers compare left to right; numeric identifiers are compared numerically, so `2.1.0-alpha.2 < 2.1.0-alpha.10`. With the same numeric core, a normal release ranks above all its pre-releases.

Build metadata follows a plus sign, as in `2.1.0+linux.4` or `2.1.0-rc.1+build.42`. SemVer ignores build metadata when determining precedence: two versions that differ only after `+` have equal precedence. Use metadata to describe how an artifact was built, not to claim that it is a higher compatible release. If deployments need to distinguish artifacts, record an immutable package reference, commit, or artifact digest in the build and provenance system.

Pre-release ordering and Composer's stability filter are separate rules. Composer recognizes `dev`, `alpha`, `beta`, `RC`, and `stable` levels. A candidate can be numerically inside a constraint and still be excluded by the project's `minimum-stability` setting. A release candidate is not selected merely because its version sorts below the final release.

## Composer Versions Are Not Only SemVer Strings

SemVer describes a version format and compatibility convention. Composer adds its own repository and constraint language around package tags and branches. For example, `v1.4.2` is commonly used as a Git tag name; the `v` is a tag prefix, and the semantic version is `1.4.2`. Composer removes that prefix when it reads a VCS tag. Internally, Composer's `composer/semver` library normalizes numeric versions to four components, so `1.4.2` can be represented internally as `1.4.2.0`. That normalized form is for comparison, not a release string to show users.

Branch names are another Composer-specific case. A branch may be required as `dev-main`, or a version-like branch such as `1.x` may be represented as `1.x-dev`. These are development references, not immutable SemVer releases. A branch head can move; do not infer release stability or public compatibility from its name. Dependency and branch resolution are covered in [Chapter 93 — Dependency Resolution](093-dependency-resolution.md), while package metadata belongs in [Chapter 94 — composer.json](094-composer-json.md).

Composer's parser is not a strict SemVer validator. For backward compatibility, it accepts some pre-release separators beyond SemVer's hyphen and supports Composer-specific version strings and branch references. When publishing a portable release, use ordinary SemVer tags; when consuming packages, apply Composer's documented normalization and stability rules. The [Composer version guide](https://getcomposer.org/doc/articles/versions.md) and [composer/semver source](https://github.com/composer/semver/blob/main/src/VersionParser.php) document this boundary.

## Composer Constraint Operators

A version in the root `require` section is usually a **constraint**, not the exact version Composer will install. Composer's operators describe a set of acceptable package versions. The operators below are Composer syntax; SemVer itself does not define caret or tilde constraints.

| Constraint | Composer interpretation | Typical use |
| --- | --- | --- |
| `1.4.2` | Exactly version `1.4.2` | Pin a version deliberately |
| `^1.4.2` | `>=1.4.2 <2.0.0` | Accept compatible `1.x` releases |
| `~1.4.2` | `>=1.4.2 <1.5.0` | Accept patch updates within the selected minor |
| `~1.4` | `>=1.4.0 <2.0.0` | Accept later minor releases before `2.0.0` |
| `1.4.*` | `>=1.4.0 <1.5.0` | Accept releases in one minor line |
| `>=1.4 <2.0` | Both comparisons must hold | State an explicit lower and upper bound |
| `^1.4 || ^2.0` | Either range may match | Support two deliberate API generations |

The caret uses the first non-zero position as its compatibility boundary: `^1.4.2` stops before `2.0.0`; `^0.4.2` stops before `0.5.0`; `^0.0.5` stops before `0.0.6`. This makes Composer's interpretation of `^` more conservative for pre-1.0 packages. For tilde constraints, `~1.4.2` allows patch releases but stops before `1.5.0`; `~1.4` allows later 1.x minor releases but stops before `2.0.0`.

Comparison constraints separated by a space or comma are AND conditions. The double pipe `||` separates alternatives. A partial hyphen range is supported too: `1.2 - 2.0` includes the `1.2` line through `2.0.*`, equivalent to `>=1.2.0 <2.1.0`; when both endpoints are complete, both endpoints are included. These expressions are parsed by Composer rather than by the SemVer specification. The official [Composer versions and constraints guide](https://getcomposer.org/doc/articles/versions.md) lists the exact forms and boundary rules.

For a library, write a constraint that permits all dependency versions the library claims to support and has tested. For an application, an illustrative requirement might be:

```json
{
  "require": {
    "monolog/monolog": "^3.0"
  }
}
```

That constraint communicates which versions the application is willing to use; it does not record the one version currently installed. The lock file records the selected package set for an application and is discussed in [Chapter 95 — composer.lock](095-composer-lock.md).

## Exact Pins, Ranges, and Stability

An exact constraint such as `1.4.2` narrows a package to that one release. It is useful when a project deliberately needs one known version or when upstream version promises cannot be trusted, but it blocks ordinary updates to `1.4.3` until the constraint itself changes. In an application, a range in `composer.json` plus a committed lock file usually separates compatibility policy from the tested installation state. Do not pin every application dependency exactly in the manifest as a substitute for a lock file.

For a reusable library, an exact pin can unnecessarily prevent consumers from combining your package with another library that needs a newer compatible release. Prefer the broadest range that matches the library's real tested support. Avoid an unbounded constraint such as `*` or `>=1.0` unless you are prepared to accept future major releases: a range that admits a release does not prove your library works with it.

Composer's root `minimum-stability` defaults to `stable`. A package-specific stability flag allows an unstable release for one requirement while leaving the root policy intact:

```json
{
  "require": {
    "acme/preview-parser": "^2.0@beta"
  },
  "minimum-stability": "stable",
  "prefer-stable": true
}
```

Here, `@beta` permits beta candidates for this requirement; it does not widen `^2.0` to unrelated majors or certify the beta as compatible. `prefer-stable` asks Composer to favor stable candidates when compatible alternatives exist. It is a preference, not a substitute for a correct constraint. See the official [Composer schema entries for stability](https://getcomposer.org/doc/04-schema.md#minimum-stability).

## Author and Consumer Practices

Package authors should:

- document the public API and supported PHP/platform versions;
- make release tags immutable and publish a new version for every change;
- classify changes from the perspective of downstream callers and implementers;
- mark public behavior as deprecated in a minor release before removing it in a later major release;
- test both the lowest and newest dependency versions allowed by declared constraints.

For example, a library can run a compatibility job using `composer update --prefer-lowest --prefer-stable` to test its lower bounds, then a normal update job to test current compatible releases. These jobs test different points in the allowed range; neither proves every combination, so choose a matrix that covers the dependencies and PHP versions that matter. Application teams should commit and test a lock-file change as a code change rather than assuming that every version admitted by the manifest is already verified. Composer's [`update` command options](https://getcomposer.org/doc/03-cli.md#update-u) document `--prefer-lowest` and `--prefer-stable`.

Consumers should read release notes for breaking changes, check which API their code uses, test the application on its target PHP/extensions, and update the lock file deliberately. A green `composer update` means Composer found a set consistent with the package metadata and policies. It does not prove that a vendor's SemVer promise was honored or that application behavior remains correct.

## Testing Compatibility Promises

Tests should cover the public contract that version numbers claim to protect. For a PHP library, include callers that use supported method signatures and types, expected return values, documented exceptions, configuration formats, and serialization or command-line formats where relevant. Add regression tests for bug fixes so a later patch release does not reintroduce the defect.

When widening a dependency constraint, run the package's tests against at least the lowest and newest supported versions and each supported PHP version. If the package supports multiple unrelated dependency generations, test each range. For applications, test the exact lock-file update in CI and on a deployment-like platform. When a breaking change is intentional, document the migration and major-version transition rather than relying on a new number to explain it.

## Common Mistakes

- Treating SemVer as a technical guarantee that every publisher follows correctly.
- Calling a release compatible without first defining the public API.
- Assuming `0.x` promises stable compatibility because a Composer caret range stays within one minor line.
- Treating a Composer constraint such as `^1.4` as SemVer syntax rather than Composer's range syntax.
- Confusing a Git tag like `v1.4.2` with the strict SemVer string `1.4.2`.
- Using build metadata after `+` to make one version sort newer than another.
- Assuming a beta is eligible for installation just because its numeric version falls inside a range.
- Pinning each dependency exactly in an application manifest instead of using a lock file to preserve the selected set.
- Publishing `*` or unbounded ranges without testing future major releases.
- Treating a successful Composer resolution as proof that package behavior is compatible.
- Adding a required method to a public PHP interface in a minor release and overlooking third-party implementers.

## Senior Engineer Thinking

Ask two separate questions: “Which versions does the publisher claim are compatible?” and “Which of those versions have we tested in this application?” SemVer and Composer constraints describe the first question. A lock file, compatibility matrix, integration tests, and release review address the second.

For an application, use the manifest constraint to express a reasonable upgrade policy and the lock file to capture the selection tested by the team. For a library, publish a consumer-friendly range only when your CI exercises its boundaries. If upstream versioning is unreliable, state that risk, constrain the range based on evidence, and plan deliberate upgrades. Version numbers reduce coordination cost only when authors and consumers treat them as part of an explicit compatibility contract.

## Exercises

1. Given `1.3.4`, classify each change as patch, minor, or major: fix a documented incorrect result; add a new class; remove a public method; add a required method to a public interface; deprecate an old method.
2. For each Composer constraint below, write an equivalent lower and upper bound and list three matching versions: `^1.4.2`, `~1.4.2`, `1.4.*`, `>=1.4 <2.0`, and `^1.4 || ^2.0`.
3. Compare `^0.4.2` with `~0.4.2`. Explain what Composer permits and why neither constraint supplies a SemVer stability guarantee for a `0.x` package.
4. Compare these SemVer versions by precedence: `2.0.0-alpha.2`, `2.0.0-alpha.10`, `2.0.0-beta`, `2.0.0-rc.1`, `2.0.0`, `2.0.0+build.7`. Explain why the final two versions have equal precedence and the role of build metadata.
5. Design a release policy for a PHP library with interfaces, configuration files, and serialized output. Define its public API and give one compatible and one breaking change for each surface.
6. Design CI jobs for a library requiring `psr/log` and supporting two PHP minor versions. Explain which dependency combinations your jobs test and what they cannot prove.
7. A package publishes `1.6.0` with a breaking method signature change. Explain why a constraint such as `^1.4` admits it and propose a consumer response that preserves a working build while the metadata issue is resolved.

## Review Questions

1. What compatibility promises do patch, minor, and major increments make after `1.0.0`?
2. Why must a package author define the public API before applying SemVer?
3. Why does `0.x` not carry the same stability expectation as `1.x`?
4. How do pre-release and build metadata affect SemVer precedence?
5. Which part of `v1.4.2` is a Git tag convention, and which part is the semantic version?
6. How does Composer's `^` range change for `0.x` versions?
7. What is the difference between an exact manifest constraint and a locked application version?
8. How do Composer's stability flags interact with `minimum-stability`?
9. Why is a successful Composer resolution not a compatibility test?
10. What does a lowest/highest dependency CI matrix establish, and what combinations may it miss?

## Summary

SemVer is a public compatibility promise: patch versions fix bugs compatibly, minor versions add compatible API or deprecations, and major versions signal breaking public API changes. Pre-releases have lower precedence than the corresponding stable version; build metadata does not change precedence. Version zero is initial development and carries no stable API guarantee. These signals are useful only when the package defines its public contract and keeps releases immutable.

Composer adds its own version parsing and constraint syntax. It strips common `v` prefixes from VCS tags and normalizes numeric versions internally, while branch names such as `dev-main` are Composer-specific development references. Composer's caret, tilde, wildcard, comparison, union, and stability rules determine candidate ranges; they do not prove that a package author followed SemVer. Applications should distinguish manifest ranges from the exact lock-file selection; libraries should declare tested compatibility and verify range boundaries in CI.

## References

- [Semantic Versioning 2.0.0 specification](https://semver.org/)
- [Composer: Versions and constraints](https://getcomposer.org/doc/articles/versions.md)
- [Composer schema: Minimum stability](https://getcomposer.org/doc/04-schema.md#minimum-stability)
- [Composer CLI: `update` options](https://getcomposer.org/doc/03-cli.md#update-u)
- [Composer Semver library](https://github.com/composer/semver)
- [Composer Semver `VersionParser` source](https://github.com/composer/semver/blob/main/src/VersionParser.php)
- [Chapter 93 — Dependency Resolution](093-dependency-resolution.md)
- [Chapter 94 — composer.json](094-composer-json.md)
- [Chapter 95 — composer.lock](095-composer-lock.md)
