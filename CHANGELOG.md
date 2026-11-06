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
  - Versioning rules within v1: `schema_version` stays `1.0.0`, and
    deprecated fields stay valid until v2. While the schema is a draft, a
    field can also move to another type
- **Email DNS posture schema** (spec/v1/schemas/email-dns.md):
  `evidencepack/email-dns@v1`, with the MX, SPF, DMARC, and DKIM records of
  each domain, an assessment of each domain, and a summary of counts and
  percentages across domains
- **TLS posture schema** (spec/v1/schemas/tls.md): `evidencepack/tls@v1`, with
  the TLS versions each endpoint accepts, its certificate, an assessment of
  each endpoint, and a summary of counts and percentages across endpoints
- **Reference types** (profiles/types/evidencepack/). `evidencepack/document@v1`
  is any document and carries no metadata fields.
  `evidencepack/identity-center-mfa-settings@v1` is a capture of the AWS IAM
  Identity Center MFA settings, dated by `captured_on`. Each posture schema now
  has a type of the same name: `cloud-posture`, `idp-posture`, `vcs-posture`,
  `endpoint-posture`, `access-posture`, `vuln-scanning`, `email-dns`, and
  `tls`
- **Endpoint posture schema** (spec/v1/schemas/endpoint-posture.md):
  `evidencepack/endpoint-posture@v1`, with fleet enrollment, device
  compliance, hardening, OS patching, and anti-malware and EDR coverage. Every
  metric is optional, and `availability` records why a metric has no reading.
  `scope.discovery_complete` is `null` when the producer could not tell
  whether it read every device
  - An `availability` reason of `unsupported` means the vendor never reports
    the metric, and `not_read` means the vendor reports it but the producer
    does not read it
- **Access posture schema** (spec/v1/schemas/access-posture.md):
  `evidencepack/access-posture@v1`, with the sign-in rules, members, admins,
  root accounts, machine credentials, and connected apps of one system.
  `system_type` is the kind of system, and `scopes` lists each account or
  subscription separately
  - A member or admin is local when they can sign in without SSO, even if
    their account is linked to SSO. Local and SSO members can overlap when the
    system does not require SSO
  - A credential is unrestricted when it has neither an IP limit nor a scope
    narrower than full access. A credential limited to chosen permissions or
    resources has a narrower scope
  - A member is inactive with no sign-in or other recorded activity in the
    last 90 days
  - With `scopes`, a top-level `sign_in` value is false when it is false in
    any scope. Otherwise it is `null` when it is `null` in any scope, and true
    when it is true in every scope
- **Vulnerability scanning schema** (spec/v1/schemas/vuln-scanning.md):
  `evidencepack/vuln-scanning@v1`, with the coverage and open findings by
  severity of each scanner in one system, and `scopes` for each account or
  subscription. A scanner turned on per resource, such as per repository, is
  `enabled` when it is on for at least one resource, and `coverage_pct` gives
  the share it covers
- The cloud, identity provider, version control, endpoint, access, and
  vulnerability scanning pages list the fields written only at the audit and
  internal levels, such as the counts. The `_present` booleans are written at
  every level
- Counts beside the percentages in the cloud posture, identity provider
  posture, and version control posture schemas, so a reader can see what each
  percentage was computed from. Each section also has a `_present` boolean that
  says whether there was anything to count
- Cloud posture `database` and `threat_detection` sections, and
  `accounts_expected` and `all_accounts_collected` at the top level
- Version control posture `repository_count`, `scanned_repository_count`, and
  `repositories_present`
- `jumpcloud` as an identity provider posture provider
- **Every artifact of a type** (spec/v1/profiles.md Sections 4.4 and 6.2). A
  clause's `artifacts` key is `any`, the default, or `all`. With `all`, every
  artifact of the clause's type must satisfy the clause, such as one
  access-posture artifact for each system. One artifact whose conditions are
  false fails the clause, even when another lacks a value the clause reads.
  Each check has counts of the artifacts evaluated and passed, and a list of
  the ones that failed. The key needs an epack-tool-validate release after
  v0.1.13

### Changed

- **Profiles companion spec** (spec/v1/profiles.md) rewritten to describe how
  the reference validator works
  - Clauses match artifacts by their manifest `schema`. The `semantic_types`
    artifact field is no longer used
  - A requirement is machine-checked, with an `id` and `satisfied_by` clauses,
    or manual. Type, freshness, severity, and conditions sit on each clause.
    Requirement-level `type`, `required`, and `cardinality` are gone, and
    profiles carry a `version`
  - Conditions read the artifact's content with JSONPath, using `eq`, `neq`,
    `gt`, `gte`, `lt`, `lte`, `exists`, `not_exists`, and `max_age_days`, with
    a `cardinality` for paths that return several values. `in`, `contains`,
    `contains_all`, `regex`, and `any` groups are gone
  - Outcomes are three-valued. A missing or null value makes a requirement
    unknown rather than failed, and clauses can grade a failure by severity
  - Overlays `skip`, `modify`, and `add` requirements by `id`, and `modify`
    replaces the whole requirement
  - Results follow the validator's output, with four warning codes. A broken
    profile or pack stops the run instead of producing findings
  - A run validates one profile. Profiles and overlays are local files whose
    digest is the SHA-256 of their bytes. The registry design moves to an
    appendix as reserved
- **Manifest reserved fields** (spec/v1/pack.md Section 10, rules.md R-028a).
  `profiles` and `overlays` hold `{source, digest}` references, as epack writes
  them, in place of the `profile` identifier, overlay identifiers, and
  `profile_lock`. The artifact `schema` field is described as the type of the
  artifact's content, which profiles match on
- The reference profiles and overlays use the new format and only published
  types. soc2-basic checks the SOC 2 report period and the penetration test
  date, and asks for the information security policy as a manual requirement.
  vendor-review adds email DNS and TLS posture from the last 90 days and asks
  for an architecture diagram. cloud-posture-overlay requires cloud, identity
  provider, and version control posture. It also requires access and
  vulnerability scanning posture for every system, with `artifacts: all`.
  hipaa-overlay tightens the `pentest` requirement to 6 months and adds the
  business associate agreement and a risk assessment as manual requirements
- control-mapping.md joins `type` entries to the one requirement whose clauses
  read that type, and its profile digest is the profile file's digest
- `account_id` is no longer required on a cloud posture account
- The email DNS posture descriptions of `enforced`, `has_reporting`, and the
  DMARC record fields follow RFC 9989. A domain's DMARC policy can come from a
  record found above it by the DNS tree walk, and `pct` is recorded for
  reference without affecting any other value
- Cloud posture field descriptions no longer use one provider's terms. Network
  exposure counts security groups with a rule open to any address
- Cloud posture Section 6 shows a condition over every account with
  `cardinality`. It no longer lists an `average` aggregation, which profiles do
  not have
- In samples/notes.md, the sample packs are built, compared, and verified with
  `epack build`, `epack diff`, and `epack verify`
- README.md and spec/README.md list the content schemas
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
- `iam`, `vuln_scanning`, and `workforce_sign_in` in the cloud posture schema,
  and `org_security` and `security_features` in the version control posture
  schema. Sign-in, members, root accounts, and machine credentials are in
  access-posture, and scanner coverage is in vuln-scanning. Producers write
  the old fields beside the new ones while profiles move, and a later
  revision of the v1 draft removes them. A profile that reads an old field
  should add its new one
  - `accounts[*].iam` `users_present`, `user_count`, `mfa_user_count`, and
    `mfa_coverage_pct` moved to access-posture's
    `members.local_members_present`, `local_member_count`,
    `local_mfa_member_count`, and `local_mfa_coverage_pct`
  - `iam.root_mfa_enabled` moved to `root.root_mfa_coverage_pct`.
    `iam.root_access_protected` is now `root_mfa_coverage_pct` at 100 with
    `root.root_access_keys_present` false
  - `iam.access_keys_present` and `access_key_user_count` moved to
    `credentials.credentials_present` and `credential_count`
  - `iam.access_key_rotation_compliant_pct` moved to
    `credentials.rotation_overdue_pct`, which counts the credentials that are
    not rotated in time
  - `iam.saml_providers_present` and `workforce_sign_in.in_use` moved to
    `sign_in.sso_configured`. `workforce_sign_in.user_count` moved to
    `members.sso_member_count`
  - `iam.rotation_compliant_user_count`, `iam.saml_provider_count`, and
    `workforce_sign_in.mfa_source` have no new field. idp-posture holds MFA for
    sign-ins through an identity provider
  - Cloud posture's `vuln_scanning` moved to a vuln-scanning scanner entry
  - Version control posture's `org_security.two_factor_required` moved to
    access-posture's `sign_in.mfa_required`
  - Version control posture's `security_features` moved to vuln-scanning
    entries for Dependabot, secret scanning, and code scanning, each with its
    `coverage_pct`
  - A system with several accounts or subscriptions lists each one in
    access-posture's and vuln-scanning's `scopes`
  - A producer that leaves access-posture or vuln-scanning out of a pack
    leaves out the old fields that moved there too, so the pack never carries
    a value its publisher chose not to share

### Removed

- Example types that no producer writes: `architecture-diagram`,
  `baa-agreement`, `cloud-config`, `iam-summary`, `idp-config`,
  `security-policy`, `vcs-config`, and `vulnerability-scan`. A policy,
  agreement, or diagram is an `evidencepack/document@v1` asked for through a
  manual requirement, and cloud, identity, and version control configuration
  are covered by the posture types
- `evidencepack/baseline@v1`. A profile needs at least one requirement

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
- **Control Mappings** (spec/v1/schemas/control-mapping.md, profiles companion)
  - `evidencepack/control-mapping@v1` embedded artifact schema: publisher-authored
    links from pack evidence to profile requirements, with no effect on
    validation outcomes
  - Evidence anchors at three granularities: whole artifact or document,
    JSON Pointer into artifact content, quoted passage or table row in a document
  - Optional stable `id` on profile requirements; mapping entries join by
    `id`, `control`, or `type`
  - `manual: true` requirement form for requirements answered by mapped
    evidence and human review
  - Optional `description` on profile requirements: prose stating what
    satisfies the requirement, echoed in validation output
  - `asserted_support` section in validation output, with mapping and manual
    warning codes
  - Worked example under examples/control-mapping/

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
