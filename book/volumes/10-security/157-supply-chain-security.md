---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 157
title: Supply Chain Security
slug: supply-chain-security
status: complete
summary: ../../_ai/chapter-summaries/157-supply-chain-security-summary.md
---

# Chapter 157 — Supply Chain Security

Software supply chain security covers the path from source code and dependencies to a running release. It includes developer machines, Git hosting, Composer, CI runners, build tools, base images, artifact storage, deployment systems, and the production environment.

Dependency security asks whether a package is safe to use. Supply chain security also asks whether the package, source revision, build inputs, artifact, and deployment identity are the ones the team intended to use.

## Why this matters

An attacker can compromise a maintainer account, publish a malicious release, alter a build script, steal a CI token, inject a command through a pull request, replace an artifact, or compromise an image used by every service. A successful attack may leave application source looking unchanged while the produced artifact is different.

Model the chain as a set of claims that must be verified:

```text
source + locked dependencies + build instructions
        → isolated build
        → identified artifact + provenance
        → verified deployment
        → observed runtime
```

Each arrow needs an identity, an integrity check, and an owner. A green test result is evidence about behavior in one environment; it is not proof that the production artifact came from the reviewed source.

## Source and review controls

Protect the default branch and release tags. Require review for workflow files, Composer manifests and lockfiles, Dockerfiles, deployment configuration, and scripts that run with credentials. Require status checks, prevent unreviewed force-pushes, and limit who can create or move release tags.

Treat pull-request data as attacker-controlled in CI. Do not interpolate titles, branch names, issue text, or untrusted filenames into shell commands. Use least-privilege, short-lived CI identities, and isolate jobs that run untrusted code from jobs that can publish or deploy.

Record the exact commit, lockfile hash, PHP version, Composer version, operating-system image, build configuration, and tool versions used to make a release. Reproducibility is useful because an independent build can compare outputs, but differences in timestamps, generated metadata, or native extensions must be understood rather than hidden.

## Build isolation and credentials

Builds should start from a controlled, reviewed environment. Pin base images by digest where practical, minimize packages and network access, and separate dependency resolution from the release job. A build that needs to download code should have only the credentials and destinations required for that step.

Do not place production secrets in a general build job. Use an identity broker or workload identity to issue a short-lived deployment credential after the job satisfies branch, review, and provenance conditions. A CI runner that can publish releases and read all production secrets is a high-impact trust boundary.

A conceptual policy might require:

```text
release may deploy only when:
  source is a protected tag
  required reviews and checks succeeded
  dependency lockfile is recorded
  artifact digest matches the build output
  provenance identifies the trusted builder
```

The syntax belongs to the CI and deployment platform; the invariant belongs to the organization. Keep the policy testable and fail closed when provenance or artifact identity is missing.

## Artifacts, provenance, and verification

Publish immutable artifacts addressed by digest. Sign artifacts or attach a verifiable attestation that identifies the source revision, builder, inputs, and build steps. Verify the signature or attestation at the deployment boundary, not only when the artifact is uploaded.

An artifact signature answers “which identity signed these bytes?” Provenance answers “how were these bytes produced?” Neither proves that the source was free of vulnerabilities, so combine them with review, dependency audits, tests, and runtime controls. Store attestations with enough retention to investigate a release months later.

Generate a software bill of materials (SBOM) for the release and connect component versions to the artifact digest. An SBOM is an inventory, not a security guarantee, but it speeds impact analysis when an advisory arrives. Keep the format and tooling consistent enough that incident responders can query it.

## Third-party and release risk

Evaluate registries, Git hosts, build images, action or plugin ecosystems, and external services as suppliers. Pin reusable CI actions and build images to reviewed versions or immutable digests. Review changes to workflow permissions and install scripts. Verify a package's namespace and provenance before adding it; a convincing name is not evidence of ownership.

Use two-person review for high-impact changes, separate approval from implementation where practical, and provide a revocation path for compromised signing or publishing identities. Monitor release and registry events, unexpected dependency changes, new outbound connections, and artifacts that differ from the expected graph.

## Deployment and runtime controls

Deploy the exact verified artifact; do not rebuild from a mutable tag during rollout. Restrict who can promote an artifact, keep rollback artifacts available, and ensure rollback does not restore a known-vulnerable dependency without an explicit decision. At runtime, use least-privilege service identities, network egress policy, read-only filesystems where practical, and resource limits.

Monitor the release identifier, artifact digest, dependency inventory, and configuration version in each process. This makes it possible to answer which instances are affected by a compromised package or build. Alert when an instance runs an unapproved digest or missing provenance.

## Incident response

If a supplier or build is compromised, stop promotion, preserve the affected artifacts and logs, revoke CI and publishing credentials, identify the first and last affected release, and compare source, lockfiles, artifacts, and deployment records. Rebuild from a known-good commit in a clean environment, rotate any credentials available to the build, and verify the new artifact at deployment.

Do not rely on a clean rebuild alone. Determine whether malicious code ran during installation or build, whether artifacts or caches were poisoned, whether production secrets were readable, and whether downstream customers received the affected artifact. Record the chain of evidence and update controls based on the root cause.

## Exercises

1. Draw the trust boundaries from a pull request to a deployed PHP container. Mark every credential and artifact identity crossing a boundary.
2. Design a release policy that requires a protected tag, Composer lockfile, artifact digest, and signed provenance. State what happens when one claim is missing.
3. Create an incident checklist for a compromised Composer package, including affected-release discovery and credential rotation.

## Review questions

- How does supply chain security extend dependency security?
- Why should CI jobs that run untrusted code not receive production secrets?
- What does an artifact signature prove, and what does provenance add?
- Why deploy a verified immutable artifact instead of rebuilding from a tag?
- How does an SBOM help during an incident without proving safety by itself?

## References

- [SLSA specification](https://slsa.dev/spec/v1.0/)
- [NIST Secure Software Development Framework (SP 800-218)](https://csrc.nist.gov/pubs/sp/800/218/final)
- [OpenSSF Scorecard](https://github.com/ossf/scorecard)
- [OWASP Software Supply Chain Security](https://owasp.org/www-project-software-supply-chain-security/)
- [in-toto Attestation Framework](https://in-toto.io/)
