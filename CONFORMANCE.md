# Evidence Pack Conformance

This document defines conformance requirements for Evidence Pack implementations and the associated test vectors.

## Terminology

- "MUST", "SHOULD", and "MAY" are to be interpreted as described in RFC 2119.
- "Spec v1" refers to the documents under `spec/v1/`.

## Conformance Levels

An implementation MAY claim conformance at one or more levels. Higher levels include all requirements of lower levels.

### Level 1: Pack Format (Consumer)

An implementation is **Level 1 Conformant** if it can:

1. Parse Evidence Packs that conform to `spec/v1/pack.md`
2. Validate path and archive requirements defined in the Pack specification
3. Verify embedded artifact digests
4. Compute `pack_digest` according to the Pack specification
5. Reject unsafe archives (at minimum: path traversal attempts and link entries such as symlinks/hardlinks) as defined by the Pack specification
6. Reject manifests that contain non-finite JSON numbers, values that do not evaluate to whole numbers where integers are required, or integer values outside 0..2^53-1

### Level 2: Attestation (Consumer)

An implementation is **Level 2 Conformant** if it meets Level 1 and can:

1. Compute `manifest_digest` using JCS canonicalization (RFC 8785) as defined in `spec/v1/attestation.md`
2. Parse and validate Sigstore bundle structure (mediaType, verificationMaterial, dsseEnvelope)
3. Verify Sigstore bundles against the public Sigstore trusted root using sigstore-go or equivalent
4. Validate in-toto statement predicateType is `https://evidencepack.dev/attestation/v1`
5. Verify subject digest matches computed manifest digest
6. Verify predicate pack_digest matches computed pack digest
7. Reject JCS inputs that contain non-finite JSON numbers

> Note: Implementations MAY support signing (producer functionality), but signing support is not required to claim Level 2 **consumer** conformance.

## Test Vectors

Conforming implementations MUST pass the applicable test vectors in `test-vectors/`.

### Vector sets by level

- **Level 1**: MUST pass all vectors under `test-vectors/pack-digest/`, `test-vectors/path-validation/`, `test-vectors/zip-safety/`, `test-vectors/structure/`, and `test-vectors/limits/`
- **Level 2**: MUST pass all Level 1 vectors plus `test-vectors/manifest-digest/`, `test-vectors/jcs/`, `test-vectors/manifest/`, and `test-vectors/attestation/`

### Running vectors (high-level)

A vector runner should:
1. Load test inputs from each vector directory
2. Execute the defined operation (verify/compute)
3. Compare actual outputs to expected outputs
4. Report pass/fail with a stable error code

## Claiming conformance

Implementations MAY claim conformance using:

> "This implementation conforms to Evidence Pack Spec v1 at Level [1|2] (consumer)."

Conformance claims SHOULD include:
1. Spec version (e.g., "Spec v1" and the release tag of this repo)
2. Conformance level (1 or 2)
3. Vector set version (from `test-vectors/VERSION` or git tag/commit)
5. Date of testing
6. Any known limitations or deviations

## Interoperability testing (recommended)

Before publicly claiming conformance, implementations SHOULD verify interoperability with at least one other independent implementation:
- Create/build in one implementation, verify in another
- Verify Sigstore bundles produced by one implementation in another

## Non-conformance

Implementations that intentionally deviate SHOULD:
1. Document deviations clearly
2. Provide a "strict" mode that enforces conformance behavior
3. Warn users when non-conformant behavior is enabled

## Updates

This document and the test vectors are versioned alongside the specification. The vector set version is recorded in `test-vectors/VERSION`.
