# AI Summary — Chapter 157 — Supply Chain Security

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains supply-chain trust boundaries from source to runtime, protected review, untrusted CI inputs, build isolation, short-lived credentials, immutable artifacts, provenance, signatures, SBOMs, supplier risk, deployment verification, incident response, exercises, and review questions.

## Concepts already explained

Supply-chain security extends dependency security to source, CI, build inputs, artifacts, and deployment. Artifact signatures identify signers; provenance describes production; neither replaces testing, review, dependency analysis, or runtime controls.

## Terminology established

Supply chain, provenance, attestation, artifact digest, immutable artifact, SBOM, protected tag, workload identity, build isolation.

## Examples used

A source-to-runtime trust chain and a conceptual deployment policy requiring reviewed source, lockfile identity, artifact digest, and trusted-builder provenance.

## Cross-references

- [Chapter 156 — Dependency Security](../../volumes/10-security/156-dependency-security.md)
- [Chapter 263 — Health Checks](../../volumes/17-production-engineering/263-health-checks.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 158 — Why Tests Exist: the Why This Matters section.

## Technical verification notes

Local links were checked in the consolidated security proofread. Supply-chain guidance links to SLSA, NIST SSDF, OpenSSF, OWASP, and in-toto.
