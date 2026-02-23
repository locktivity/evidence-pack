# Evidence Pack Specification

**Version:** 1.0
**Status:** Draft

## Scope

Evidence Pack defines formats for Evidence Packs and attestations. Out of scope: authentication mechanisms, identity verification, and retention/archiving policies.

## Notational Conventions

RFC 2119 and RFC 8174 keywords apply as defined in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

## Specifications

### Core Specifications (Normative)

| Document | Description | Status |
|----------|-------------|--------|
| [Evidence Pack Format](v1/pack.md) | Archive structure, manifest schema, artifact index | Draft |
| [Attestation Format](v1/attestation.md) | Sigstore signing and verification | Draft |
| [Specification Rules](v1/rules.md) | Consolidated normative requirements | Draft |

### Companion Specifications

| Document | Description | Status |
|----------|-------------|--------|
| [Profiles & Semantic Validation](v1/profiles.md) | Types, profiles, overlays, and semantic validation | Draft |

### Implementation Guidance (Informative)

| Document | Description | Status |
|----------|-------------|--------|
| [Security Appendix](v1/security-appendix.md) | Implementation checklist, threat model, code patterns | Draft |

## Quickstart Flow

### Publisher Flow

1. Collect evidence artifacts
2. Build `pack.zip` with `manifest.json`
3. Compute `pack_digest`
4. Sign manifest with Sigstore (OIDC → Fulcio certificate → Rekor log)
5. Include `.sigstore.json` bundle in `attestations/`
6. Distribute pack (HTTP, S3, etc.)

### Consumer Flow

1. Download `pack.zip`
2. Verify embedded artifact digests and `pack_digest`
3. Verify Sigstore attestation against trusted root
4. Verify signer identity matches expected (if configured)
5. Accept or reject the pack based on policy

```
Publisher --> pack.zip + sigstore bundle --> Distribution
Consumer  --> pack.zip --> verify --> decision
```

## Compatibility

Implementations SHOULD state which core specifications they implement (pack format, attestation format) and the spec version (for example, "Evidence Pack v1.0 pack + attestation").

## Error Codes (Recommended)

Implementations SHOULD emit stable error codes for common failures. Error codes are for UX/testing and MUST NOT be used to infer security properties. At minimum:

- **JSON/JCS:** `invalid_json`, `jcs_invalid`, `duplicate_keys`, `non_finite_number`
- **Manifest:** `missing_required_field`, `unsupported_spec_version`, `invalid_timestamp`, `invalid_digest_format`
- **Digest:** `pack_digest_mismatch`, `manifest_digest_mismatch`
- **Paths/ZIP:** `invalid_path`, `reserved_name`, `duplicate_path`, `extra_artifact_in_zip`, `zip_symlink`, `zip_bomb`
- **Attestations:** `invalid_bundle_media_type`, `missing_verification_material`, `missing_dsse_envelope`, `signature_invalid`, `certificate_identity_mismatch`

## Common Pitfalls

- **Digest ordering:** `pack_digest` sorting is byte-wise on UTF-8, not locale collation.
- **Timestamps:** Follow the specification's timestamp profile exactly; do not assume RFC3339 features like fractional seconds or offsets are accepted unless explicitly allowed.
- **Paths:** No backslashes, no `..`, no empty segments; UTF-8 only.
- **Extra files:** Any file under `artifacts/` must be listed in the manifest.
- **Attestation filenames:** Use `.sigstore.json` extension for Sigstore bundles.
- **Signer identity:** Sigstore bundle validity does not imply trusted signer; configure expected identities.
- **ZIP metadata:** Ignore timestamps/comments; only paths and bytes matter.

## Glossary

| Term | Definition |
|------|------------|
| Evidence Pack | ZIP archive containing `manifest.json`, `artifacts/`, and optional `attestations/`. |
| Manifest | `manifest.json` at the archive root with `spec_version`, `stream`, `generated_at`, `pack_digest`, source metadata, and the artifact index. |
| Artifact | Entry in the manifest artifact index representing a piece of evidence data. |
| Embedded artifact | Artifact whose bytes are stored inside the archive (`type: "embedded"`). |
| Referenced artifact | Artifact whose bytes are stored externally (`type: "reference"`). |
| Pack digest (`pack_digest`) | SHA-256 digest computed from the canonical list of embedded artifact paths and digests. |
| Manifest digest (`manifest_digest`) | SHA-256 digest of JCS-canonicalized `manifest.json`. |
| Sigstore bundle | JSON file containing certificate chain, transparency log entry, and signed DSSE envelope. |
| in-toto statement | Attestation payload format with subject (manifest.json) and predicate (pack metadata). |
| Provenance | Manifest field describing source packs for merged packs. |
| Embedded attestation | Sigstore bundle from a source pack, included in merged pack provenance. |

## License

This specification is licensed under [Apache 2.0](../LICENSE).
