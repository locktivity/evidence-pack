# Evidence Pack Governance

## Overview

Evidence Pack is an open specification for packaging and distributing security evidence. This document defines how the specification evolves, how releases are made, and what stability guarantees implementations can rely on.

## Principles

1. **Stability First**  
   The specification prioritizes stability and backwards compatibility. Breaking changes require strong justification and an explicit migration plan.

2. **Security by Design**  
   Changes are evaluated for security implications. Where feasible, the specification prefers deterministic and safe-by-default behavior.

3. **Implementation Feedback**  
   Changes are informed by real-world implementation experience. Proposed behavioral changes should be validated against test vectors and at least one reference implementation.

4. **Interoperability**  
   The goal is independent implementations that interoperate. Ambiguities are resolved in favor of deterministic behavior.

## Normative sources

Conformance is defined by:
- the normative text in `spec/v1/`, and
- the conformance test vectors in `test-vectors/`.

Tooling implementations (e.g., verifiers/builders) are non-normative; they must follow the spec and vectors.

## Versioning and stability guarantees

The specification uses Semantic Versioning:

- **Major versions** (e.g., 2.0.0): breaking changes that may require implementation updates.
- **Minor versions** (e.g., 1.1.0): backwards-compatible additions.
- **Patch versions** (e.g., 1.0.1): clarifications, errata, and non-breaking fixes.

### Compatibility contract (v1)

Within a major version line (v1.x.y):
- Packs that conform to Spec v1 MUST remain valid across all v1 minor/patch releases.
- New fields MUST be optional and ignorable by older implementations unless explicitly stated otherwise.
- Deterministic algorithms (digests, canonicalization, path rules) MUST NOT change in a way that changes outcomes for existing conformant packs, except in a major version.

### Patch release rule

Patch releases MUST NOT change pass/fail outcomes for existing conformance vectors, except to correct an error in the vectors themselves or to fix a security vulnerability that requires stricter validation (which must be documented prominently in the release notes).

## Change process

### Proposing changes

1. **Open an issue** describing the problem, proposed solution, and compatibility impact.
2. **Discussion** to gather feedback and alternatives.
3. **Draft PR** with concrete spec text changes.
4. **Review** including security considerations and interoperability impact.
5. **Approval and merge** by maintainers.

### Requirements for behavioral changes

Any change that affects verifier behavior MUST include:
- updates to relevant test vectors (or new vectors), and
- documentation of expected behavior changes in the changelog/release notes.

### Security issues

Security issues should be reported privately via `SECURITY.md`. Security fixes may be fast-tracked. When feasible, fixes should be accompanied by new test vectors that prevent regressions.

### Breaking changes

Breaking changes require:
1. Clear justification
2. A migration plan (including guidance for producers and consumers)
3. A deprecation story (when applicable)
4. Security review

## Maintainers

Maintainers are responsible for:
- reviewing and merging changes
- managing releases and tags
- ensuring spec/vectors remain consistent
- handling security reports

Current maintainers:
- Locktivity team (GitHub org: @locktivity)

Maintainership changes (adding/removing maintainers) are proposed via PR and approved by existing maintainers.

## Community

- **GitHub Issues**: bug reports and change proposals
- **GitHub Discussions**: questions and design discussions
- **Pull Requests**: specification, vectors, and documentation changes

All participants are expected to follow the Contributor Covenant Code of Conduct.

## Licensing

The specification and conformance materials in this repository are licensed under Apache 2.0 (see `LICENSE`). Implementations may use any license.
