# Evidence Pack Test Vector Runner Spec (v1.0)

This document defines a minimal, language-agnostic output contract for running the test vectors in this repository.

## Goals

- Ensure consistent pass/fail results across implementations
- Provide stable error codes for interoperability
- Allow optional diagnostic data without breaking compatibility

## Required Output Schema

Each vector execution MUST produce a JSON object with the following fields:

```json
{
  "vector": "path/to/vector.json",
  "valid": true,
  "result": {
    "ok": true,
    "errors": [
      {"code": "error_code", "message": "optional", "location": "optional"}
    ],
    "computed": {
      "pack_digest": "optional",
      "manifest_digest": "optional",
      "jcs_canonical": "optional",
      "jcs_payload": "optional",
      "canonical_input": "optional"
    }
  }
}
```

### Field Notes

- `vector`: relative path to the vector file
- `valid`: whether the vector is expected to pass
- `result.ok`: actual pass/fail result
- `errors`: MAY be empty for passing vectors; for invalid vectors, at least one error is RECOMMENDED
- `computed`: include values when the vector defines an `expected` entry for that output

## Exit Codes

- `0` = all vectors passed
- `1` = one or more vectors failed

## Stable Error Codes (recommended)

Implementations SHOULD use the following codes where applicable:

### JSON / JCS
- `invalid_json`
- `jcs_invalid`
- `duplicate_keys`
- `non_finite_number`
- `invalid_number`

### Manifest / Pack
- `missing_required_field`
- `invalid_timestamp`
- `invalid_digest_format`
- `pack_digest_mismatch`
- `manifest_digest_mismatch`

### Paths / ZIP
- `invalid_path`
- `path_not_nfc` — path not in Unicode NFC form (R-317, R-318)
- `reserved_name`
- `path_traversal`
- `duplicate_path`
- `directory_slash_mismatch` — directory attributes inconsistent with trailing slash (R-323, R-347)
- `invalid_directory_entry` — directory entry with nonzero size (R-322)
- `zip_symlink`
- `zip_special_file`
- `zip_bomb`
- `artifact_too_large` — single artifact exceeds size limit
- `pack_too_large` — cumulative extraction exceeds pack size limit

### Attestations (Sigstore)
- `invalid_bundle_media_type` — bundle mediaType is not a valid Sigstore bundle type
- `missing_verification_material` — bundle missing verificationMaterial
- `missing_dsse_envelope` — bundle missing dsseEnvelope
- `invalid_certificate_chain` — certificate chain validation failed
- `invalid_tlog_entry` — transparency log entry validation failed
- `signature_invalid` — DSSE signature verification failed
- `certificate_identity_mismatch` — certificate identity does not match expected signer

## Failure Precedence (Recommended)

Test vectors MAY specify `expected_error`. When multiple failures apply, implementations SHOULD report the expected error when it is applicable, even if other errors are also present. This keeps conformance results consistent and avoids masking intent (for example, `key_revoked` should be reported even if a signature is also invalid).

## Versioning

The test vector set version is recorded in `test-vectors/VERSION`. Runner implementations SHOULD surface that value in their summary output.

### Version Bump Policy (recommended)

- **Add or update vectors:** increment `test-vectors/VERSION` (e.g., `1.0` → `1.1`).
- **Spec minor/major changes with incompatible vectors:** add a new versioned directory (e.g., `test-vectors/v1.1/`) and keep the root `VERSION` pointing to the latest set.
- **Conformance claims:** include both the spec version and the vector set version.
