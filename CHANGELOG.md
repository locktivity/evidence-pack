# Changelog

All notable changes to the Evidence Pack specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Cloud posture schema** (spec/v1/schemas/cloud-posture.md)
  - `logging.audit_log_enabled`: an account-wide audit log of management
    activity is enabled
  - `logging.audit_log_all_regions`: that audit log covers every region the
    account uses, true where the provider's audit log is global
  - `iam.user_count`: the number of human user identities that
    `iam.mfa_coverage_pct` counts
  - `iam.access_key_user_count`: the number of users with an active
    long-lived programmatic credential.
    `iam.access_key_rotation_compliant_pct` counts these users
  - `iam.saml_provider_count`: the number of SAML identity providers the
    account trusts directly for sign-in, not counting one created by a
    workforce sign-in service such as AWS IAM Identity Center
  - `workforce_sign_in`: a central sign-in service that signs people in to the
    accounts, such as AWS IAM Identity Center. It has `in_use`, `user_count`,
    and `mfa_source` (`sign_in_service`, `external_idp`, or `unknown`), and is
    left out when the collector does not report one
  - Versioning rules within v1: `schema_version` stays `1.0.0`, and
    deprecated fields stay valid until v2
- **Email DNS posture schema** (spec/v1/schemas/email-dns.md):
  `evidencepack/email-dns@v1`, with the MX, SPF, DMARC, and DKIM records of
  each domain, an assessment of each domain, and a summary of counts and
  percentages across domains
- **TLS posture schema** (spec/v1/schemas/tls.md): `evidencepack/tls@v1`, with
  the TLS versions each endpoint accepts, its certificate, an assessment of
  each endpoint, and a summary of counts and percentages across endpoints

### Changed

- Cloud posture field descriptions no longer use one provider's terms: user MFA
  coverage counts human user identities that can sign in, root fields describe
  the highest-privilege owner identity, key rotation covers long-lived
  programmatic credentials, and network exposure counts network rules open to
  any address
- `iam.access_key_rotation_compliant_pct` in the cloud posture schema: the
  percentage of users with an active long-lived programmatic credential whose
  active credentials are all rotated within policy, rather than the
  percentage of credentials rotated
- The cloud posture, identity provider posture, and version control posture
  schemas allow `null` on every metric and metric group, for a value the
  collector could not read. Each page lists the fields that are required
- `user_security` and its four metrics are no longer required in the identity
  provider posture schema, so only `schema_version`, `collected_at`,
  `provider`, and `org_domain` are. `schema_version` and each posture section
  in that schema have a description

### Deprecated

- `logging.cloudtrail_enabled` and `logging.cloudtrail_multiregion` in the cloud
  posture schema: aliases of `logging.audit_log_enabled` and
  `logging.audit_log_all_regions` with the same values. Producers emit both
  names until v2, which removes the aliases

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
