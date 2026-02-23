# Changelog

All notable changes to the Evidence Pack specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-01-25

### Added

- Initial release of the Evidence Pack specification
- **Pack Format** (spec/v1/pack.md)
  - ZIP archive structure with manifest.json, artifacts/, attestations/
  - Embedded and referenced artifact support
  - Pack digest computation algorithm
  - Path validation rules
  - Size and safety limits
- **Attestation Format** (spec/v1/attestation.md)
  - Sigstore integration (Fulcio certificates, Rekor transparency log)
  - JCS canonicalization (RFC 8785)
  - Manifest digest computation
  - in-toto statement format with Evidence Pack predicate
  - Multiple attestation support (multiple signers)
- Test vectors for conformance testing
- JCS canonicalization constraints and integer range validation guidance
- Clarified registry trust pinning policies (strict vs TOFU)
- Split ZIP safety requirements into mandatory core and optional hardening profile
- Relaxed path character restrictions to UTF-8 with safety-focused exclusions
- Added a hardened registry conformance level (Level 3H)
- Clarified registry URL safety baseline vs cross-origin hardening
- Added path normalization and attestation-exclusion test vectors
- Added test vector runner spec and explicit vector set versioning
- Sample packs demonstrating evidence evolution
- Governance and conformance documentation

### Security
- Archive safety requirements (e.g., path traversal prevention; link/file-type rejection) as specified
- Guidance for resource limits (e.g., maximum sizes / counts) to mitigate malicious packs
- Sigstore-based identity verification for signed packs

---

## Version History

| Version | Date | Status |
|---------|------|--------|
| 1.0.0 | 2026-01-25 | Current |

## Migration Guides

Migration guides will be provided here when breaking changes occur between major versions.
