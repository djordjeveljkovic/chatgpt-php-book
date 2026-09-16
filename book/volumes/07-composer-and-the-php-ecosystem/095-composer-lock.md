---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 95
title: composer.lock
slug: composer-lock
status: complete
summary: ../../_ai/chapter-summaries/095-composer-lock-summary.md
---

# Chapter 95 — composer.lock

## Why This File Matters

Two developers can start with the same `composer.json` and, at different times, install different package versions that satisfy its constraints. That can change behavior even when neither developer changed application code. In a deployed application, an unreviewed package change can arrive during a build and make a release behave differently from the one that passed CI.

The `composer.lock` file records a concrete dependency selection for a project. It lets Composer install the package versions selected during an intentional resolution, instead of choosing a fresh set every time. For an application, that makes the dependency tree part of the reviewed source state. For a reusable library, its own lock file has a different role: it can help maintainers repeat their development setup, but it does not pin the library’s dependencies for the applications that consume it.

A lock file improves repeatability; it does not make every input to a release immutable. It does not contain PHP itself, install required extensions, guarantee a repository will remain available, or prove that third-party code is safe. Learning its boundary helps teams use it as a reliable build input without treating it as a complete deployment image.

## Mental Model

Separate the manifest’s compatibility policy, the lock file’s selected package set, and the installed files:

```text
composer.json constraints + repository/platform inputs
                         ↓ intentional resolution
                  composer.lock
                         ↓ composer install
           vendor/ files + generated autoloader
                         ↓
              application runs on PHP
```

`composer.json` describes requirements and configuration; see [Chapter 94](094-composer-json.md). A deliberate `update` resolves those inputs and writes the selected package information to the lock file. `install` uses the lock file to materialize that selection locally. Chapter [93](093-dependency-resolution.md) explains the resolution process and the separate roles of `install` and `update`; this chapter focuses on what the lock records and how teams should handle it.

The lock file is generated data. Let Composer maintain it. Editing package records by hand is fragile because the file contains a graph of packages and metadata that must agree, not just a list of preferred version strings.

## Applications and Reusable Libraries

### Applications should commit the lock file

An application is the final consumer of its dependency tree. Commit both `composer.json` and `composer.lock` so that developers, CI, and deployment use the same selected set. After checkout, the normal setup command is:

```sh
composer install
```

When a lock file is present, `install` uses its recorded package selections. A production build can omit development packages while installing the same locked application dependencies:

```sh
composer install --no-dev --no-interaction --optimize-autoloader
```

`--no-dev` changes which packages are installed for that build; it does not mean that production should make a new dependency selection. Test the production install mode as part of the release pipeline. The exact lock-file and generated `vendor/` tree should match the application revision being promoted.

### Libraries publish constraints, not a downstream lock

A library’s `composer.json` describes compatible dependencies so that Composer can combine it with the rest of a consumer’s application. The library maintainer may keep and commit a lock file to make the library’s own tests repeatable. Composer ignores that lock when installing the library as a dependency in another root project: the consuming project’s lock file owns the resolved set for that application.

This distinction changes testing strategy. A library’s committed lock proves what its maintainers tested in one selected environment, not every dependency version its declared constraints permit. Test supported PHP versions and dependency combinations separately. The official [Composer library guidance](https://getcomposer.org/doc/02-libraries.md#lock-file) describes the lock file’s limited role for libraries.

## What the Lock File Records

The exact JSON content varies with the Composer version and the project’s dependencies. Common generated fields include:

| Lock data | What it describes | What it does not mean |
| --- | --- | --- |
| `packages` | Selected non-dev package records | A copy of each package’s source tree |
| `packages-dev` | Selected development-only package records when included | A separate dependency-resolution universe for production |
| Package `name` and `version` | The selected package identity and version | A proof that its code behaves as expected |
| `source` metadata | Where source can be fetched and the recorded VCS reference | A guarantee that every remote host remains available |
| `dist` metadata | Archive type and URL, and reference or checksum fields when supplied | A checksum guarantee when the checksum field is empty or absent |
| `platform` and `platform-dev` | Root PHP/extension requirements for runtime and development | The actual PHP binary or extensions on a deployment host |
| `platform-overrides` | Platform values used to model a target when configured | A change to the machine running Composer or the application |
| `content-hash` | Whether dependency-relevant manifest content matches this lock | A cryptographic signature over installed package files |
| Stability, alias, and plugin API metadata | Additional data needed to preserve the generated package state | A substitute for reviewing or pinning the Composer build tool |

The package arrays contain package records with metadata such as dependency requirements, autoload information, package type, and repository download/source data. The lock file stores references to obtain package content; it does not embed every archive or PHP file. Where both source and distribution metadata exist, Composer’s normal preference is to install from a distribution archive, although project configuration or a command can prefer source. A lock can therefore describe a VCS reference and a downloadable archive separately.

Treat this as a field guide, not a schema to hand-author. Composer may add fields across versions, and older lock files may lack current metadata. The [Composer lock implementation](https://github.com/composer/composer/blob/main/src/Composer/Package/Locker.php) shows what current Composer writes; the official [repository documentation](https://getcomposer.org/doc/05-repositories.md) explains package source and distribution metadata.

### Package versions and references

Each selected package record includes a normalized or pretty package version and may include both `source` and `dist` sections. The source reference commonly identifies the VCS commit or tag from which the package can be checked out. The dist entry identifies an archive URL and may carry a corresponding reference and `shasum` field. Package repositories provide this metadata as part of the package version record.

This distinction matters when diagnosing a build. If the recorded commit is no longer reachable, source installation can fail. If an archive URL has gone away or a mirror changed, distribution installation can fail even though the lock file still parses. The presence of a reference does not automatically mean that the downloaded archive is authenticated by a nonempty checksum; inspect the actual metadata and supply-chain controls for the packages that matter.

## The Content Hash and Freshness

The `content-hash` helps Composer determine whether dependency-relevant parts of `composer.json` still correspond to the lock file. When someone changes a requirement, repository, or modeled platform without updating the lock, validation can report that the lock is out of date. This catches a common class of accidental mismatch before an install or release proceeds.

It is not a hash of every byte in `composer.json`. Changes to descriptive or workflow fields do not necessarily alter package resolution and may not change the hash. Nor is it a checksum over `vendor/`, the contents of downloaded archives, or the PHP files in a package. It detects lock/manifest freshness for the inputs Composer treats as relevant; it does not establish artifact integrity or author authenticity. Composer’s current [`Locker::getContentHash()` implementation](https://github.com/composer/composer/blob/main/src/Composer/Package/Locker.php) selects the relevant manifest data before computing the value.

Use validation to check the relationship:

```sh
composer validate --strict
```

The `validate` command checks manifest syntax/schema and, when a lock exists, whether it is current with the manifest. `--strict` makes warnings fail the command as well as errors. Use it before committing dependency changes and in CI. See the [Composer CLI reference](https://getcomposer.org/doc/03-cli.md#validate) for current options.

## Making and Reviewing a Dependency Change

The lock file should change as part of an intentional dependency update. For example, a maintainer may add a package, then inspect both files:

```sh
composer require psr/log
git diff -- composer.json composer.lock
composer validate --strict
```

`require` may update the manifest and immediately resolve/install the new dependency set. The diff shows the declared change and its resolved consequence. A broader update can move many packages allowed by their constraints, so review each unexpected package movement rather than assuming every line is incidental. For resolution diagnostics and update scope, use [Chapter 93](093-dependency-resolution.md).

Before merging a dependency change, a reviewer should be able to answer:

1. Which direct requirement or environment change motivated it?
2. Which packages changed, and are indirect changes expected?
3. Are PHP and extension requirements still satisfied in each target environment?
4. Did scripts, plugins, repositories, or package provenance change?
5. Did CI install from a clean checkout and run application tests against the new set?
6. Did the production install mode succeed without development packages?

The lock diff is useful because it makes a package update visible, but it is not self-explanatory. Inspect package names and versions, source references, new transitive dependencies, removals, and any changed download metadata. Combine the diff with package changelogs, advisories, provenance review, and tests.

## Resolving Lock-File Merge Conflicts

Do not manually combine arbitrary hunks of two lock files and assume the result is valid. Two branches may have added different dependencies, changed overlapping constraints, or locked the same package to different versions. The text merge can lose the exact choices made on one branch or leave packages missing from or unneeded by the manifest.

A safer workflow is:

1. Resolve `composer.json` first so it states the intended combined requirements.
2. Inspect both branches’ lock diffs to understand what each change introduced.
3. Restore a known-good lock file as the base, usually from the target branch.
4. Run a targeted `composer update` for the affected package set, allowing related packages to move only when needed.
5. Validate the result and inspect the newly generated lock diff.
6. Install from the result and run the test suite in the target-like platform.

If Git cleanly merges the package records and only the lock’s content hash conflicts, `composer update --lock` may be enough to refresh the hash and package metadata such as mirrors or URLs without deliberately selecting new package versions. This narrow case commonly occurs when branches add or update different packages with no overlapping or conflicting dependencies. Before using the command, confirm that the merged lock already contains a valid selection for the combined manifest; it is not a general conflict resolver. Composer’s [merge-conflict guide](https://getcomposer.org/doc/articles/resolving-merge-conflicts.md) explains this case and why other textual merges can lose package selections. If uncertain, return to a known-good lock and regenerate it from the resolved manifest rather than editing package entries by hand.

## Reproducibility and Its Limits

For an application, a committed lock lets repeated installs start from the same selected versions and references. Composer documents that identical installs normally create equivalent `vendor/` contents, apart from file timestamps. This is a major improvement over fetching unconstrained current releases during every build.

The lock still does not freeze the entire build environment:

- **PHP and extensions:** The lock can record requirements and platform overrides, but the real runtime must provide them. Run `composer check-platform-reqs` in the deployment image; it checks the real platform rather than trusting `config.platform`. [Chapter 93](093-dependency-resolution.md) discusses this boundary.
- **Composer itself:** Composer’s version and configuration can affect lock interpretation, scripts, plugin APIs, and generated autoload files. Use a controlled Composer release in CI and deployment when exact build behavior matters.
- **Repositories and artifacts:** Install still needs package content from a local cache, mirror, or repository. Availability, credentials, and archive retention remain operational dependencies.
- **Scripts and plugins:** Project scripts and approved plugins can run code or modify files during installation. A fixed package set does not make those actions harmless or deterministic.
- **Application and infrastructure inputs:** Environment variables, system libraries, PHP configuration, generated assets, database state, and deployment configuration remain outside the lock.
- **Behavior and security:** A lock says which package versions were selected; it does not prove the application is compatible with them or that they contain no vulnerability or malicious code.

For a higher-assurance build, combine the lock with a pinned runtime/container image, controlled Composer version, trusted artifact sources, audited dependencies, and CI tests. Promote the built artifact rather than resolving new dependencies on the live host. A lock file is one important build input, not the complete artifact or a substitute for provenance checks.

## Useful Commands and Diagnostics

Use each command for its particular question:

```sh
# Check manifest and lock consistency, including warnings.
composer validate --strict

# Inspect package versions recorded in the lock file.
composer show --locked

# Preview what installation would do without applying operations.
composer install --dry-run

# Check actual PHP/extensions in the environment that will run the app.
composer check-platform-reqs --no-dev
```

`show --locked` helps inspect the selected package set even when `vendor/` is absent or stale. `install --dry-run` is a preview, not a substitute for a clean install test. `validate` checks manifest/lock consistency; it does not prove the download endpoint is reachable, package code works, or the target platform is configured. `check-platform-reqs` answers the latter platform question using the actual interpreter and extensions.

If Composer reports that the manifest and lock disagree, first determine whether a dependency-relevant input changed. If so, use an intentional update and review the resulting diff. If only non-resolution metadata changed, do not modify package versions just to make the warning disappear; use the appropriate narrow Composer command only after confirming it is safe. Avoid deleting the lock as a first troubleshooting step: a fresh resolution can select a materially different set.

## Common Mistakes

- **Not committing an application lock file.** Teammates and deployment can resolve different allowed versions at different times.
- **Expecting a library’s lock file to control consumers.** The consumer application’s root lock owns its installed graph.
- **Treating `content-hash` as an artifact checksum.** It compares relevant manifest content with lock state; it does not hash package files.
- **Assuming the lock embeds package archives.** It stores package metadata and references; artifacts still must be available.
- **Manually editing version entries.** Use Composer commands to preserve dependency and metadata consistency.
- **Resolving conflicts by accepting whichever lock hunk Git keeps.** Reconstruct the combined state from the resolved manifest and review the generated result.
- **Using `composer update` during every deployment.** That reselects dependencies instead of materializing the reviewed application set.
- **Assuming lock-backed install validates production PHP.** Check the actual PHP version and extensions in the target image.
- **Treating an unchanged lock as proof of an unchanged build.** Composer version, repositories, artifacts, scripts, plugins, and runtime inputs still matter.

## Senior Engineer Thinking

Treat every lock-file change as a reviewable statement about the software that will run. Ask why the package set changed, whether the selected references are available from trusted sources, how CI exercised the new set, and how quickly the team can respond if a dependency has a vulnerability.

For an application, the lock file helps align developer workstations, tests, and deploys. It only does so if all those stages actually install from the same committed revision and use comparable PHP platforms. For a library, the published compatibility ranges are the consumer contract; maintainers need a test matrix that examines more than the single graph pinned locally.

When reproducing a past incident, retain more than `composer.lock`: keep the release artifact or image digest, source revision, Composer/runtime versions, build logs, and relevant configuration. The lock helps answer “which package versions?”; operational records answer “which complete build and environment?”

## Exercises

1. In a small application, install a direct package and inspect the resulting lock record. Identify its version, source reference, distribution URL, and any checksum field. Explain which values are package metadata and which identify a retrieval location.
2. Delete only `vendor/` and run `composer install` from a clean environment. Compare the selected package versions with the lock file and explain what information is recreated.
3. Add a `require-dev` tool, update the lock, then compare a normal install with `composer install --no-dev`. Which packages are present in the lock, and which files are installed in each mode?
4. Change a dependency constraint without updating the lock. Run `composer validate` and explain what the content hash tells you and what it does not tell you.
5. Create two branches that add non-conflicting direct dependencies. Merge their manifests and lock files, then follow Composer’s official conflict guidance. Compare a targeted regeneration with `composer update --lock` and explain why the latter applies only in the content-hash-only case.
6. For a library, commit a lock file and then install that library into a different root application. Observe which lock file governs the resulting package set.
7. List three inputs a committed lock file cannot freeze in your deployment system and choose a separate control for each.

## Review Questions

1. Which artifact describes dependency compatibility, and which records a concrete selected package set?
2. Why should an application normally commit `composer.lock`?
3. When can a library maintainer use a lock file, and why does it not pin downstream consumers?
4. What do `packages`, `packages-dev`, `platform`, `platform-dev`, and `platform-overrides` describe?
5. What is the `content-hash` designed to detect? What does it not validate?
6. How are package source metadata and distribution metadata different?
7. Why can an install still fail even if `composer.lock` is valid and committed?
8. When might `composer update --lock` help with a merge conflict, and why should it not be used as a general repair?
9. Which commands check manifest/lock freshness, inspect locked packages, preview installation, and verify the real runtime platform?

## Summary

`composer.lock` records the package versions and retrieval metadata selected for a project. Applications normally commit it and use `composer install` to materialize the reviewed set. Libraries may use a lock for their own development, but consuming applications resolve their own package set from their root manifest and lock.

The file includes package and platform information, references for source or distribution, and a content hash that helps Composer detect relevant manifest changes. It does not embed package code, guarantee every artifact is available or authenticated, or freeze the runtime and build process. Keep it generated by Composer, review its diffs, resolve merge conflicts from the intended manifest, and validate and test the exact install mode used in deployment.

## References

- [Composer basic usage — commit and install from a lock file](https://getcomposer.org/doc/01-basic-usage.md#installing-dependencies)
- [Composer libraries — lock-file practice](https://getcomposer.org/doc/02-libraries.md#lock-file)
- [Composer CLI — install and update](https://getcomposer.org/doc/03-cli.md#install-i)
- [Composer CLI — validate](https://getcomposer.org/doc/03-cli.md#validate)
- [Composer repositories and package metadata](https://getcomposer.org/doc/05-repositories.md)
- [Resolving Composer lock-file merge conflicts](https://getcomposer.org/doc/articles/resolving-merge-conflicts.md)
- [Composer `Locker` source](https://github.com/composer/composer/blob/main/src/Composer/Package/Locker.php)
- [Chapter 93 — Dependency Resolution](093-dependency-resolution.md)
- [Chapter 94 — `composer.json`](094-composer-json.md)
- [Chapter 96 — Semantic Versioning](096-semantic-versioning.md)
