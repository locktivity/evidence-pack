# Evidence Pack Test Vectors

This directory contains test vectors for validating Evidence Pack implementations.

The vector set version is recorded in `test-vectors/VERSION`.

## Directory Structure

```
test-vectors/
├── manifest-digest/         # Manifest digest computation tests
├── pack-digest/            # Pack digest computation tests
├── attestation/            # Sigstore attestation tests
├── path-validation/        # Path validation tests
├── zip-safety/             # ZIP safety tests
├── jcs/                    # JCS canonicalization tests
├── manifest/               # Manifest field validation tests
├── structure/              # Pack structure validation tests
└── limits/                 # Size and count limit tests
```

## Test Vector Format

The canonical runner output and vector format expectations are defined in `test-vectors/runner-spec.md`. Vectors may contain single-case fields (`input`, `expected`, `valid`) or multi-case `tests[]` arrays as described there.

## Requirements

Conforming implementations MUST pass ALL test vectors marked as `valid: true` and MUST reject ALL test vectors marked as `valid: false`.

Vectors MAY include a `profile` field. Vectors marked `profile: "hardening"` are RECOMMENDED but are not required for base conformance.

For digest-related vectors, `expected` SHOULD include both the canonical bytes (or canonical string) and the expected SHA-256 digest.

For runner output requirements and error code guidance, see `test-vectors/runner-spec.md`.

## Running Tests

Implementations should:

1. For each test vector file:
   - Parse the input
   - Apply the operation (digest computation, validation, etc.)
   - Compare with expected output
   - Report pass/fail

## Test Categories

### manifest-digest/

Tests for JCS canonicalization and manifest digest computation per spec Section 4.1.2.

### pack-digest/

Tests for pack digest computation per spec Section 5.1:
- Canonical list format: `{path}\t{digest}\n`
- Byte-wise sorting
- Empty artifact list handling
- Exclusion of attestation entries from digest computation

### attestation/

Tests for Sigstore bundle structure validation:
- Valid bundle structure (mediaType, verificationMaterial, dsseEnvelope)
- Invalid or missing mediaType
- Missing required bundle components
- in-toto statement predicateType validation

Note: Cryptographic verification of Sigstore bundles is delegated to sigstore-go and verifies against the public Sigstore trusted root.

### path-validation/

Tests for path validation per spec Section 2.3:
- Character set restrictions
- Length limits
- Reserved names
- Structural rules
- Unicode NFC normalization (R-317, R-318)
- Windows paths: drive letters and UNC paths (R-345, R-346)

### zip-safety/

Tests for ZIP archive safety:
- Symlink rejection
- Path traversal attempts
- Duplicate path detection
- Compression ratio limits
- Directory entry consistency (R-321–R-323, R-347)
- Byte accounting for extraction limits (R-324, R-325)
- Hard link handling guidance (R-326, R-327)

### jcs/

Tests for JCS (JSON Canonicalization Scheme) per RFC 8785:
- Key sorting
- Number normalization
- Non-finite number rejection
- Duplicate key rejection (R-319, R-320, R-334, R-335)
- String escaping

### manifest/

Tests for manifest field validation:
- Required fields
- Timestamp formats
- Digest formats
- Provenance validation (Sigstore embedded attestations)
- Integer validation

### structure/

Tests for pack structure validation:
- ZIP format requirements
- Directory layout
- Attestation file placement

### limits/

Tests for size and count limits:
- Minimum enforceable limits
- Artifact size limits

## Sigstore Verification

Sigstore bundle verification uses the sigstore-go library against the public Sigstore trusted root (Fulcio CA, Rekor public key). Cryptographic verification test vectors are not included as sigstore-go has comprehensive test coverage.

Test vectors in this repository focus on:
- Bundle structure validation (mediaType, required fields)
- in-toto statement validation (predicateType, subject)
- Manifest and pack digest verification
