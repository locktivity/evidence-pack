# Profiles & Semantic Validation

**Version:** 1.0
**Status:** Draft

> **Draft Specification:** This companion specification is under active development. The concepts and requirements defined here may change based on implementation feedback.

## 1. Introduction

A structurally valid Evidence Pack tells you nothing about whether it contains the *right* evidence. A pack could pass all integrity checks but be missing a required SOC 2 report, or include a pentest summary without the full findings.

Profiles solve this by declaring what a pack must contain.

### 1.1 Relationship to Core Spec

This spec builds on the reserved fields defined in [pack.md](pack.md) Section 10:

- `profile` — References the profile this pack claims to conform to
- `overlays` — Additional constraints applied on top of the profile
- `profile_lock` — Pinned digests for reproducible validation

**Core spec validators ignore these fields.** A Level 1 validator will accept a pack regardless of whether it satisfies its declared profile. Semantic validation requires a profile-aware validator implementing this companion spec.

### 1.2 Design Rationale

The core spec focuses on structural correctness and cryptographic integrity—stable, well-understood problems. Semantic validation is domain-specific and evolving. By keeping them separate, the core spec can remain stable while profiles iterate based on real-world usage.

### 1.3 Artifact Semantic Type Declaration

For semantic validation to work, artifacts must declare their semantic type. This companion spec defines a new optional artifact field:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `semantic_type` | String | No | Type identifier (e.g., `evidencepack/soc2-report@v1`) |

An artifact matches a profile requirement when `artifact.semantic_type == requirement.type`.

**Interoperability:** The `semantic_type` field is defined by this companion spec, not the core spec. Adding `semantic_type` is backwards-compatible with validators that implement only the core spec, because they only need to parse and validate the fields defined by the core spec and can ignore additional JSON object members. Semantic validators implementing this spec MUST also accept unknown artifact fields they do not recognize (P-021).

**Naming rationale:** The field is named `semantic_type` (not `type_id`) to clearly distinguish it from the core spec's `type` field ("embedded"/"reference") and to indicate that it carries semantic meaning for profile validation.

## 2. Core Concepts

| Concept | Description |
|---------|-------------|
| **Type** | A reusable definition for a category of evidence artifact (e.g., "SOC 2 Type II Report") |
| **Profile** | A named contract declaring what types a pack must include |
| **Overlay** | Additive constraints that modify a profile for specific contexts |

### 2.1 How They Compose

```json
{
  "spec_version": "1.0",
  "profile": "evidencepack/soc2-basic@v1",
  "overlays": ["evidencepack/hipaa-overlay@v1"],
  "profile_lock": [
    { "id": "evidencepack/soc2-basic@v1", "digest": "sha256:abc123..." },
    { "id": "evidencepack/hipaa-overlay@v1", "digest": "sha256:def456..." },
    { "id": "evidencepack/soc2-report@v1", "digest": "sha256:789def..." }
  ],
  "artifacts": [
    {
      "type": "embedded",
      "path": "artifacts/soc2-2025.pdf",
      "digest": "sha256:...",
      "size": 1024000,
      "content_type": "application/pdf",
      "semantic_type": "evidencepack/soc2-report@v1",
      "collected_at": "2025-03-15T10:00:00Z",
      "metadata": {
        "report_period_start": "2024-01-01",
        "report_period_end": "2024-12-31",
        "auditor": "Big4 LLP"
      }
    }
  ]
}
```

Validation proceeds in layers:

1. **Resolve** the profile, overlays, and all referenced types (verify digests if locked)
2. **Merge** overlays onto the base profile to produce an effective profile
3. **Match** artifacts to requirements by `artifact.semantic_type == requirement.type`
4. **Validate** each artifact's metadata against its type's `metadata_schema`
5. **Check** cardinality and freshness constraints
6. **Report** findings (pass, fail, or warning for each requirement)

## 3. Types

A type is a reusable definition for a category of evidence artifact. It specifies what metadata the artifact should provide and how to validate it.

### 3.1 Type Definition

```json
{
  "id": "evidencepack/soc2-report@v1",
  "name": "SOC 2 Type II Report",
  "description": "Annual SOC 2 Type II attestation report from an accredited auditor",
  "content_types": ["application/pdf"],
  "metadata_schema": {
    "type": "object",
    "properties": {
      "report_period_start": { "type": "string", "format": "date" },
      "report_period_end": { "type": "string", "format": "date" },
      "auditor": { "type": "string" },
      "trust_services_criteria": {
        "type": "array",
        "items": { "enum": ["security", "availability", "processing_integrity", "confidentiality", "privacy"] }
      }
    },
    "required": ["report_period_start", "report_period_end", "auditor"]
  },
  "controls": ["SOC2-CC1.1", "SOC2-CC1.2"]
}
```

### 3.2 Type Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Unique identifier in `namespace/name@vN` format |
| `name` | String | Yes | Human-readable name |
| `description` | String | No | Extended description of what this type represents |
| `content_types` | Array | No | Allowed MIME types for artifacts of this type (case-insensitive comparison) |
| `metadata_schema` | Object | No | JSON Schema for validating `artifact.metadata` (see below) |
| `controls` | Array | No | Control framework mappings this type addresses |
| `examples` | Array | No | Conformance examples for collector testing (see Section 3.5) |

**What `metadata_schema` validates:** The schema validates the `metadata` object on the artifact entry, not the file contents (e.g., PDF bytes). If future versions support extracting metadata from file contents, that will be a separate mechanism.

**MIME type comparison:** Validators MUST compare `content_type` values case-insensitively (per RFC 2045). Before comparison, validators SHOULD trim leading and trailing ASCII whitespace from both values. For example, `application/PDF` matches `application/pdf`.

**JSON Schema version:** Validators MUST support JSON Schema draft-07 at minimum. Type definitions MAY include a `$schema` field to declare which draft the schema uses (e.g., `"$schema": "http://json-schema.org/draft-07/schema#"`). If `$schema` is absent, validators SHOULD assume draft-07.

**Format keyword enforcement:** Validators MUST enforce the `format` keyword for the following values: `date`, `date-time`, `email`, `uri`. Other format values are treated as annotations (validation does not fail on unknown formats). This ensures consistent validation across implementations for common formats while allowing extensibility.

### 3.3 Identifier Format

Type, profile, and overlay identifiers MUST follow this format:

```
Format: namespace/name@vN
```

Where:
- `namespace`: lowercase ASCII alphanumeric, hyphens, underscores; 1-63 characters; no leading/trailing hyphens
- `name`: lowercase ASCII alphanumeric, hyphens, underscores; 1-63 characters; no leading/trailing hyphens
- `vN`: literal `v` followed by a positive integer with no leading zeros

**Regex:** `^[a-z0-9][a-z0-9_-]{0,62}/[a-z0-9][a-z0-9_-]{0,62}@v[1-9][0-9]*$`

**Examples:**
- Valid: `evidencepack/soc2-report@v1`, `acme/internal-policy@v2`, `my_org/custom_type@v10`
- Invalid: `SOC2-Report@v1` (uppercase), `acme/policy@latest` (non-numeric), `acme/policy@v01` (leading zero)

### 3.4 Resolution

Types, profiles, and overlays are resolved from configured registries or local cache. The `resolve()` function returns a wrapper object containing the definition and its precomputed digest.

**Registry URL structure:**

```
GET https://registry.example.com/types/{namespace}/{name}@v{N}
GET https://registry.example.com/profiles/{namespace}/{name}@v{N}
GET https://registry.example.com/overlays/{namespace}/{name}@v{N}
```

**Example request:**

```
GET https://schemas.evidencepack.dev/types/evidencepack/soc2-report@v1
```

**Response format:**

```json
{
  "id": "evidencepack/soc2-report@v1",
  "digest": "sha256:789def...",
  "definition": { ... }
}
```

Registries MUST return the same response structure for types, profiles, and overlays. The `definition` field contains the type, profile, or overlay document as defined in Sections 3, 4, and 5 respectively.

**Resolution return type:**

```
resolve(id) -> { id, digest, definition } | null
```

- `id`: The requested identifier
- `digest`: SHA-256 digest of the JCS-canonicalized `definition` bytes
- `definition`: The type/profile/overlay document itself

Validators use `digest` for lock verification and `definition` for validation logic. The digest is provided by the registry response (or computed locally for cached/bundled definitions) — it is not a property embedded in the definition document itself.

**Digest verification:** Validators SHOULD compute the digest locally from the returned `definition` bytes and confirm it equals the `digest` in the registry response. This guards against registry bugs or tampering.

Resolution rules are defined in Section 7.

### 3.5 Type Examples

Types MAY include an `examples` array for collector conformance testing. Each example provides a sample input (mocked API response) and expected output (valid artifact), enabling automated verification that collector implementations produce correct output.

**Example structure:**

```json
{
  "id": "evidencepack/vcs-config@v1",
  "name": "Version Control Configuration",
  "examples": [
    {
      "name": "github-org",
      "description": "GitHub organization with mixed branch protection",
      "input_url": "https://schemas.evidencepack.dev/examples/vcs-config/github-org/input.json",
      "output_url": "https://schemas.evidencepack.dev/examples/vcs-config/github-org/output.json"
    }
  ]
}
```

**Example fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | String | Yes | Short identifier for this example (lowercase, alphanumeric, hyphens) |
| `description` | String | No | Human-readable description of what scenario this example represents |
| `input_url` | String | Yes | URL to fetch sample API response (collector input) |
| `output_url` | String | Yes | URL to fetch expected artifact output |

**Conformance testing:** Collectors are conformant for a type when they produce output matching the expected output for all examples. The conformance test procedure is:

1. Fetch the example input from `input_url`
2. Run the collector with the input
3. Validate the collector's output against the type's `metadata_schema`
4. Compare the collector's output to the expected output from `output_url`

## 4. Profiles

A profile is a named contract that declares what a pack must contain. Profiles reference types and specify requirements (required vs optional, cardinality, freshness).

### 4.1 Profile Structure

```json
{
  "id": "evidencepack/soc2-basic@v1",
  "name": "SOC 2 Basic",
  "description": "Minimum evidence for a SOC 2 security review",
  "requirements": [
    {
      "type": "evidencepack/soc2-report@v1",
      "required": true,
      "cardinality": { "min": 1, "max": 1 },
      "freshness": { "max_age_days": 365 }
    },
    {
      "type": "evidencepack/pentest-report@v1",
      "required": true,
      "cardinality": { "min": 1 },
      "freshness": { "max_age_days": 365 }
    },
    {
      "type": "evidencepack/security-policy@v1",
      "required": false
    }
  ]
}
```

### 4.2 Requirement Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | String | — | Type identifier this requirement references (required) |
| `required` | Boolean | `true` | Syntactic sugar for `cardinality.min` (see below) |
| `cardinality.min` | Integer | `1` if required, `0` otherwise | Minimum artifact count |
| `cardinality.max` | Integer | unlimited | Maximum artifact count |
| `freshness.max_age_days` | Integer | unlimited | Maximum days since `collected_at` |

**`required` is syntactic sugar:** The `required` field only affects the default value of `cardinality.min` when `cardinality.min` is absent. If `cardinality.min` is explicitly set, it takes precedence. Validation uses the effective `minCount` to determine whether artifacts are missing:
- `minCount = cardinality.min` (if present), otherwise `1` if `required` is true, otherwise `0`
- Emit `MISSING_REQUIRED` when `minCount > 0` and artifact count is `0`
- Emit `CARDINALITY_VIOLATION` when artifact count is greater than `0` but less than `minCount`, or greater than `maxCount`

### 4.3 Requirement Type Uniqueness

Within a profile's `requirements` array, each `type` value MUST be unique. Duplicate type entries produce ambiguity during validation and overlay merging.

Similarly, an overlay's `add_requirements` MUST NOT introduce a type that already exists in the effective profile at the time of application. If violated, validators MUST emit `DUPLICATE_REQUIREMENT_TYPE` as an error.

## 5. Overlays

Overlays are additive constraints that modify a base profile. They're useful for expressing cross-cutting requirements (like HIPAA) that apply on top of existing profiles.

### 5.1 Overlay Structure

```json
{
  "id": "evidencepack/hipaa-overlay@v1",
  "name": "HIPAA Overlay",
  "description": "Additional requirements for HIPAA compliance",
  "add_requirements": [
    {
      "type": "evidencepack/baa-agreement@v1",
      "required": true
    },
    {
      "type": "evidencepack/hipaa-risk-assessment@v1",
      "required": true,
      "freshness": { "max_age_days": 365 }
    }
  ],
  "modify_requirements": [
    {
      "type": "evidencepack/pentest-report@v1",
      "freshness": { "max_age_days": 180 }
    }
  ]
}
```

### 5.2 Merge Semantics

Overlays are applied in declared order. For each overlay:

1. **`add_requirements`** — Append new requirements to the effective profile
2. **`modify_requirements`** — Update existing requirements matched by `type` in the effective profile *at application time*

**Modification rules:**
- Specified fields override the effective requirement's fields
- Unspecified fields are preserved from the effective requirement
- If multiple overlays modify the same requirement, later overlays win
- `modify_requirements` targets the effective profile, not the original base profile. This means overlay B can modify a requirement added by overlay A (if B is applied after A).

**Merge error handling:**

- For each entry in `add_requirements`: if the type already exists in the effective profile, emit `DUPLICATE_REQUIREMENT_TYPE` and do not add the requirement. Continue processing remaining entries.
- For each entry in `modify_requirements`: if the target type is not present in the effective profile, emit `OVERLAY_TARGET_NOT_FOUND` and skip that modification. Continue processing remaining entries.

**Deep merge for nested objects:** When an overlay specifies a nested object like `freshness` or `cardinality`, it performs a shallow merge at that level. For example, if an overlay specifies `freshness: { max_age_days: 180 }`, it replaces only `max_age_days` while preserving any other `freshness` subfields that may exist in the base requirement (if any are added in future versions).

**`required` in overlays:** Overlays MUST NOT set the `required` field in `modify_requirements`. To change the minimum artifact count, set `cardinality.min` explicitly. Overlays MAY set `required` in `add_requirements` (it affects the default `cardinality.min` for the new requirement).

**Why additive only?** Overlays cannot remove requirements from a profile. This ensures that applying an overlay never weakens the overall requirements. If you need a weaker profile, create a new base profile.

## 6. Semantic Validation

Semantic validation checks whether a pack satisfies its declared profile. This runs after structural validation (core spec) succeeds.

### 6.1 Artifact Matching

An artifact matches a requirement when:

```
artifact.semantic_type == requirement.type
```

Artifacts without a `semantic_type` field are not matched to any requirement and produce an `UNTYPED_ARTIFACT` warning.

### 6.2 Freshness Source

Freshness is calculated from the artifact's `collected_at` field:

- `collected_at` MUST use the exact format `YYYY-MM-DDTHH:MM:SSZ` (consistent with core spec timestamps)
- Timezone offsets and fractional seconds are not permitted; only the `Z` suffix is allowed
- If `collected_at` is missing and freshness is required, emit `MISSING_COLLECTED_AT` error
- If `collected_at` is present but malformed, emit `INVALID_COLLECTED_AT` error
- Parse `collected_at` as UTC and compute age in whole days: `floor((now - collected_at) / 86400)`

**Current time (`now`):** The validator's current time in UTC. Validators MUST use UTC for freshness calculations to ensure consistent results regardless of local timezone.

**Example:** `2025-03-15T10:00:00Z`

### 6.3 Validation Procedure

Semantic validation proceeds as follows:

1. **Build lock index** — If `profile_lock` is absent, treat it as an empty list. Index entries by `id`. On duplicate, emit `DUPLICATE_LOCK_ENTRY` per P-017 and retain the first digest (do not overwrite).

2. **Resolve profile** — Resolve per Section 7 and verify locks per P-002. Emit `PROFILE_NOT_FOUND` or `DIGEST_MISMATCH` as appropriate.

3. **Resolve and merge overlays** — If `overlays` is absent, treat it as an empty list. For each overlay in declared order, resolve and verify locks. Apply merge semantics per Section 5.2. Emit `OVERLAY_NOT_FOUND`, `DIGEST_MISMATCH`, or `OVERLAY_TARGET_NOT_FOUND` as appropriate.

4. **Resolve types** — For each type referenced by the effective profile, resolve and verify locks. Emit `TYPE_NOT_FOUND` or `DIGEST_MISMATCH` as appropriate.

5. **Check unpinned dependencies** — For the profile, all overlays, and all types in the effective profile, emit `UNPINNED_DEPENDENCY` warning if not in `profile_lock`. Validators SHOULD emit at most one `UNPINNED_DEPENDENCY` per identifier per validation run.

6. **Index artifacts** — Group artifacts by `semantic_type`. Emit `UNTYPED_ARTIFACT` warning for artifacts without `semantic_type`. Emit `INVALID_SEMANTIC_TYPE` warning for artifacts whose `semantic_type` does not match the identifier format (Section 3.3).

7. **Check requirements** — For each requirement in the effective profile:
   - Check cardinality (always, even if type unavailable)
   - If type unavailable, skip all checks that depend on the type definition (`content_type`, `metadata_schema`) per P-022. Cardinality and freshness still run.
   - For each matching artifact: check freshness (P-010/P-011/P-027), `content_type` (P-019), and `metadata` (P-012/P-018/P-024)

8. **Check unknown artifacts** — Emit `UNKNOWN_ARTIFACT` warning for typed artifacts not matching any requirement.

See Appendix A for an informative reference implementation.

### 6.4 Findings Format

Validation produces a list of findings. Each finding has a severity, code, message, and context:

```json
{
  "findings": [
    {
      "severity": "error",
      "code": "MISSING_REQUIRED",
      "message": "Missing required artifact of type evidencepack/soc2-report@v1",
      "type": "evidencepack/soc2-report@v1"
    },
    {
      "severity": "error",
      "code": "STALE_ARTIFACT",
      "message": "Artifact is 400 days old, max allowed is 365",
      "path": "artifacts/pentest-2024.pdf",
      "type": "evidencepack/pentest-report@v1"
    },
    {
      "severity": "warning",
      "code": "UNPINNED_DEPENDENCY",
      "message": "Dependency not pinned in profile_lock",
      "id": "evidencepack/soc2-report@v1"
    }
  ],
  "summary": {
    "errors": 2,
    "warnings": 1,
    "passed": false
  }
}
```

**Pass/fail semantics:** Validation passes if and only if there are zero error-severity findings. Any error finding (including early errors like `DUPLICATE_LOCK_ENTRY` or `PROFILE_NOT_FOUND`) means `summary.passed = false`. Validators MAY return early when the profile cannot be resolved or fails lock verification, because subsequent steps depend on it. Otherwise, validators SHOULD continue collecting findings after encountering errors to provide complete diagnostics.

**Finding context fields:** Findings SHOULD include `type` for requirement-scoped issues (e.g., `MISSING_REQUIRED`, `CARDINALITY_VIOLATION`) and `path` for artifact-specific issues (e.g., `STALE_ARTIFACT`, `SCHEMA_VIOLATION`). Artifact-level findings SHOULD include both `path` and `type` to provide complete context for diagnostics.

**Findings ordering:** Validators SHOULD emit findings in a stable order to facilitate testing and debugging. Recommended order: lock errors, resolution errors (profile, overlays, types), overlay merge errors, unpinned dependency warnings, untyped artifact warnings, then requirement checks in profile order, then artifact checks in artifact index order, then unknown artifact warnings.

### 6.5 Finding Codes

| Code | Severity | Description |
|------|----------|-------------|
| `MISSING_REQUIRED` | error | A required type has no matching artifacts |
| `CARDINALITY_VIOLATION` | error | Too few or too many artifacts for a type |
| `STALE_ARTIFACT` | error | Artifact exceeds freshness requirement |
| `MISSING_COLLECTED_AT` | error | Artifact missing `collected_at` when freshness required |
| `INVALID_COLLECTED_AT` | error | Artifact `collected_at` present but not in required format |
| `SCHEMA_VIOLATION` | error | Artifact metadata fails type's `metadata_schema` |
| `MISSING_METADATA` | error | Artifact missing `metadata` when type specifies `metadata_schema` |
| `CONTENT_TYPE_MISMATCH` | error | Artifact `content_type` not in type's `content_types` |
| `MISSING_CONTENT_TYPE` | error | Artifact missing `content_type` when type specifies `content_types` |
| `PROFILE_NOT_FOUND` | error | Could not resolve declared profile |
| `OVERLAY_NOT_FOUND` | error | Could not resolve declared overlay |
| `TYPE_NOT_FOUND` | error | Could not resolve type referenced by requirement |
| `OVERLAY_TARGET_NOT_FOUND` | error | Overlay modifies a type not present in the effective profile at application time |
| `DUPLICATE_REQUIREMENT_TYPE` | error | Profile or overlay adds a requirement for a type that already exists in the effective profile |
| `DIGEST_MISMATCH` | error | Resolved digest doesn't match `profile_lock` entry |
| `DUPLICATE_LOCK_ENTRY` | error | Same identifier appears multiple times in `profile_lock` |
| `UNPINNED_DEPENDENCY` | warning | Dependency not pinned in `profile_lock` |
| `UNTYPED_ARTIFACT` | warning | Artifact has no `semantic_type` field |
| `INVALID_SEMANTIC_TYPE` | warning | Artifact's `semantic_type` is present but does not match the identifier format |
| `UNKNOWN_ARTIFACT` | warning | Typed artifact's `semantic_type` not in any profile requirement |

## 7. Resolution and Registry

### 7.1 Resolution Priority

When resolving profiles, overlays, or types:

1. Look up a pinned digest in `profile_lock` (if present)
2. Use local cache if available (and if pinned, verify digest matches)
3. Fetch from configured registry if permitted (and if pinned, verify digest matches)

### 7.2 Network Policy

Validators MUST NOT automatically fetch from remote registries without explicit consent. This follows the same pattern as the core spec's external URI handling.

**Explicit consent** means one of:
- A configuration file that enables network access (e.g., `network: enabled`)
- A command-line flag that enables network access (e.g., `--network` or `--online`)
- An interactive prompt where the user confirms network access

The presence of a URL in the pack (such as a profile identifier containing a registry hostname) does NOT constitute consent.

**Offline mode:** Validators SHOULD support offline validation using:
- Pre-configured local definitions bundled with the validator
- Cached definitions from previous fetches
- Definitions embedded in a configuration file

**Online mode:** When network access is enabled, validators:
- MUST use HTTPS only
- MUST validate TLS certificates
- SHOULD cache fetched definitions by digest
- MUST re-verify cached definitions against `profile_lock` digests

### 7.3 Registry Discovery

Registries are configured explicitly. This spec does not define automatic registry discovery via `.well-known` (that may be added in a future version).

```yaml
# Example validator configuration
registries:
  - url: https://schemas.evidencepack.dev
    trusted: true
  - url: https://internal.acme.com/evidence-pack
    trusted: true

offline_definitions:
  - path: /etc/evidence-pack/types/
```

**Trust configuration:** The trust model for profile registries is separate from the trust model for pack attestations (core spec). A future version may introduce signed profile/type definitions using Sigstore. Until then, trust is established by explicit registry configuration and digest verification via `profile_lock`.

### 7.4 Digest Computation

Digests in `profile_lock` and registry responses are computed over the canonical JSON representation of the definition document using JCS (RFC 8785).

**Digest format:** `sha256:<hex-lowercase>`

**Computation steps:**
1. Parse the definition JSON (profile, overlay, or type)
2. Extract the `definition` object (if wrapped in registry response)
3. Canonicalize using JCS (RFC 8785)
4. Compute SHA-256 over the canonical bytes
5. Encode as lowercase hex prefixed with `sha256:`

This ensures the same definition always produces the same digest regardless of whitespace or key ordering in the source.

**Hex normalization:** Validators MUST treat hex digits as case-insensitive when reading digests (accepting both `sha256:abc...` and `sha256:ABC...`), but MUST emit lowercase hex when writing digests. This avoids spurious mismatches due to case differences.

### 7.5 Locking for Reproducibility

For reproducible validation, `profile_lock` SHOULD include entries for:

- The declared profile
- All declared overlays
- All types referenced by the effective profile (after overlay merging)

Each entry in `profile_lock` MUST have a unique `id`. Duplicate entries produce a `DUPLICATE_LOCK_ENTRY` error.

If `profile_lock` is present but incomplete, validators emit `UNPINNED_DEPENDENCY` warnings for missing entries.

**Strict mode:** Validators MAY support a strict mode where `UNPINNED_DEPENDENCY` is elevated to an error.

## 8. Reference Profiles

These reference profiles demonstrate the concepts and can be used as starting points. They are published under the `evidencepack/` namespace.

| Profile | Description |
|---------|-------------|
| `evidencepack/soc2-basic@v1` | Minimum evidence for a SOC 2 security review. Requires SOC 2 Type II report and penetration test summary. |
| `evidencepack/soc2-comprehensive@v1` | Extended SOC 2 evidence including security policies, incident response plan, and architecture diagrams. |
| `evidencepack/hipaa-overlay@v1` | HIPAA-specific overlay. Adds BAA agreement and risk assessment requirements. |

## 9. Test Vectors

> **TODO:** Test vector files are not yet available. The table below defines the conformance expectations; actual `.zip` files will be published in `test-vectors/profiles/` before the spec leaves Draft status.

Conforming implementations MUST pass the following test vectors:

| Vector | Description | Expected Result |
|--------|-------------|-----------------|
| `valid-soc2-basic.zip` | Pack satisfying evidencepack/soc2-basic@v1 | PASS |
| `missing-pentest.zip` | Pack declaring soc2-basic but missing pentest report | MISSING_REQUIRED |
| `stale-soc2.zip` | Pack with SOC 2 report older than 365 days | STALE_ARTIFACT |
| `missing-collected-at.zip` | Pack with artifact missing `collected_at` | MISSING_COLLECTED_AT |
| `invalid-collected-at.zip` | Pack with malformed `collected_at` (e.g., offset instead of Z) | INVALID_COLLECTED_AT |
| `digest-mismatch.zip` | Pack with `profile_lock` that doesn't match resolved profile | DIGEST_MISMATCH |
| `unpinned-type.zip` | Pack with incomplete `profile_lock` | UNPINNED_DEPENDENCY (warning) |
| `overlay-target-missing.zip` | Overlay modifies type not in base profile | OVERLAY_TARGET_NOT_FOUND |
| `hipaa-overlay-valid.zip` | Pack with soc2-basic + hipaa-overlay satisfying both | PASS |
| `untyped-artifact.zip` | Pack with artifact missing `semantic_type` field | UNTYPED_ARTIFACT (warning) |

## 10. Requirements Summary

| ID | Level | Requirement |
|----|-------|-------------|
| P-001 | MUST | Validators MUST resolve profiles before validation |
| P-002 | MUST | Validators MUST verify `profile_lock` digests before using resolved content |
| P-003 | MUST NOT | Validators MUST NOT silently substitute identifier versions |
| P-004 | MUST | Validators MUST fail explicitly if resolution fails |
| P-005 | MUST | Overlays MUST be applied in declared order |
| P-006 | MUST NOT | Overlays MUST NOT remove requirements from profiles |
| P-007 | MUST | Identifiers MUST follow `namespace/name@vN` format |
| P-008 | MUST | Findings MUST include severity, code, and descriptive message |
| P-009 | MUST | Artifact type matching MUST use exact `artifact.semantic_type == requirement.type` |
| P-010 | MUST | `collected_at` MUST use the exact format `YYYY-MM-DDTHH:MM:SSZ` (no offsets or fractional seconds) |
| P-011 | MUST | If freshness required and `collected_at` missing, emit `MISSING_COLLECTED_AT` |
| P-012 | MUST | `metadata_schema` MUST validate `artifact.metadata`, not file contents |
| P-013 | MUST | If overlay modifies type not in effective profile, emit `OVERLAY_TARGET_NOT_FOUND` |
| P-014 | MUST NOT | Validators MUST NOT fetch from remote registries without explicit consent |
| P-015 | SHOULD | `profile_lock` SHOULD include all dependencies for reproducible validation |
| P-016 | SHOULD | Validators SHOULD emit `UNPINNED_DEPENDENCY` for unlocked dependencies |
| P-017 | MUST | Each entry in `profile_lock` MUST have a unique `id` |
| P-018 | MUST | If type specifies `metadata_schema` and artifact lacks `metadata`, emit `MISSING_METADATA` |
| P-019 | MUST | If type specifies `content_types`, emit `MISSING_CONTENT_TYPE` if artifact lacks `content_type`, or `CONTENT_TYPE_MISMATCH` if artifact's `content_type` is not in the allowed list |
| P-020 | MUST | Digests MUST be computed over JCS-canonicalized (RFC 8785) definition bytes |
| P-021 | MUST | Semantic validators MUST accept unknown artifact fields they do not recognize |
| P-022 | MUST | If a type is unavailable (not found or digest mismatch), skip schema and content_type checks for that type |
| P-023 | MUST | Validators MUST treat digest hex as case-insensitive on input, but emit lowercase on output |
| P-024 | MUST | Validators MUST support JSON Schema draft-07 at minimum for `metadata_schema` validation |
| P-025 | MUST NOT | The presence of a URL in the pack MUST NOT constitute consent for network access |
| P-026 | MUST | Validation MUST pass if and only if there are zero error-severity findings |
| P-027 | MUST | If `collected_at` present but malformed, emit `INVALID_COLLECTED_AT` |
| P-028 | MUST | Within a profile's `requirements` array, each `type` value MUST be unique |
| P-029 | MUST | An overlay's `add_requirements` MUST NOT introduce a type already present in the effective profile |
| P-030 | SHOULD | Validators SHOULD emit at most one `UNPINNED_DEPENDENCY` per identifier per validation run |
| P-031 | SHOULD | Validators SHOULD compute the digest locally from fetched definitions and verify it matches the registry-provided digest |
| P-032 | MUST | Validators MUST use UTC for freshness calculations |

---

## Appendix A: Reference Implementation (Informative)

> **Note:** This appendix is informative, not normative. The pseudocode below illustrates one way to implement the validation procedure from Section 6.3. Conformance is determined by the requirements in Section 10, not by matching this implementation.

```
// Helper: resolve and verify lock in one step
function resolveLocked(id, lockedDigests, notFoundCode):
  wrapper = resolve(id)  // returns { id, digest, definition } or null
  if not wrapper:
    return { ok: false, wrapper: null, err: error(notFoundCode, id) }
  // Normalize digest to lowercase per P-023 before comparing
  normalizedDigest = lowercase(wrapper.digest)
  if id in lockedDigests and normalizedDigest != lockedDigests[id]:
    return { ok: false, wrapper: null, err: error(DIGEST_MISMATCH, id) }
  return { ok: true, wrapper: wrapper, err: null }

// Helper: timestamp format validation
function isValidTimestamp(s):
  return matches(s, "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$")

function validateSemantic(pack):
  findings = []
  lockedDigests = {}  // id -> digest from profile_lock

  // Step 1: Build lock index (P-017)
  // Normalize digests to lowercase per P-023
  profileLock = pack.profile_lock or []  // treat absent as empty list
  for lock in profileLock:
    normalizedDigest = lowercase(lock.digest)
    if lock.id in lockedDigests:
      // Do not overwrite first entry; record both digests for diagnostics
      findings.add(error(DUPLICATE_LOCK_ENTRY, lock.id, { first: lockedDigests[lock.id], second: normalizedDigest }))
    else:
      lockedDigests[lock.id] = normalizedDigest

  // Step 2: Resolve profile (P-001, P-002, P-004)
  // Early exit: profile resolution is required for all subsequent steps
  result = resolveLocked(pack.profile, lockedDigests, PROFILE_NOT_FOUND)
  if not result.ok:
    findings.add(result.err)
    return findings  // preserve any earlier findings (e.g., DUPLICATE_LOCK_ENTRY)
  profileWrapper = result.wrapper

  // Step 3: Resolve and merge overlays (P-002, P-005, P-013)
  effectiveProfile = profileWrapper.definition
  overlays = pack.overlays or []  // treat absent as empty list
  for overlayId in overlays:
    result = resolveLocked(overlayId, lockedDigests, OVERLAY_NOT_FOUND)
    if not result.ok:
      findings.add(result.err)
      continue
    effectiveProfile = merge(effectiveProfile, result.wrapper.definition, findings)

  // Step 4: Resolve types (P-002, P-004)
  typeWrappers = {}  // type id -> { id, digest, definition }
  for req in effectiveProfile.requirements:
    result = resolveLocked(req.type, lockedDigests, TYPE_NOT_FOUND)
    if not result.ok:
      findings.add(result.err)
      continue
    typeWrappers[req.type] = result.wrapper

  // Step 5: Check unpinned dependencies (P-015, P-016)
  // Use a set to emit at most one UNPINNED_DEPENDENCY per id
  allDeps = [pack.profile] + overlays + [req.type for req in effectiveProfile.requirements]
  warnedUnpinned = set()
  for dep in allDeps:
    if dep not in lockedDigests and dep not in warnedUnpinned:
      findings.add(warning(UNPINNED_DEPENDENCY, dep))
      warnedUnpinned.add(dep)

  // Step 6: Index artifacts by semantic_type
  artifactsByType = {}
  for artifact in pack.artifacts:
    if artifact.semantic_type:
      artifactsByType[artifact.semantic_type].append(artifact)
    else:
      findings.add(warning(UNTYPED_ARTIFACT, artifact.path))

  // Step 7: Check requirements (P-009, P-018, P-019, P-022)
  for req in effectiveProfile.requirements:
    matchingArtifacts = artifactsByType.get(req.type, [])
    typeWrapper = typeWrappers.get(req.type)

    // Cardinality (always checked, even if type unavailable)
    // minCount is authoritative; required only affects the default when cardinality.min is absent
    minCount = req.cardinality?.min ?? (1 if req.required else 0)
    maxCount = req.cardinality?.max or infinity
    count = len(matchingArtifacts)
    if minCount > 0 and count == 0:
      findings.add(error(MISSING_REQUIRED, req.type))
    else if count < minCount:
      findings.add(error(CARDINALITY_VIOLATION, req.type))
    if count > maxCount:
      findings.add(error(CARDINALITY_VIOLATION, req.type))

    // Freshness (always checked, even if type unavailable) (P-010, P-011, P-027)
    if req.freshness?.max_age_days:
      for artifact in matchingArtifacts:
        if not artifact.collected_at:
          findings.add(error(MISSING_COLLECTED_AT, artifact.path, req.type))
        else if not isValidTimestamp(artifact.collected_at):
          findings.add(error(INVALID_COLLECTED_AT, artifact.path, req.type))
        else:
          age = daysSince(artifact.collected_at)  // collected_at is UTC Z format per P-010
          if age > req.freshness.max_age_days:
            findings.add(error(STALE_ARTIFACT, artifact.path, req.type))

    // Skip type-based checks if type unavailable (P-022)
    if not typeWrapper:
      continue

    typeDef = typeWrapper.definition
    for artifact in matchingArtifacts:
      // Content type (P-019)
      if typeDef.content_types:
        if not artifact.content_type:
          findings.add(error(MISSING_CONTENT_TYPE, artifact.path, req.type))
        else if artifact.content_type not in typeDef.content_types:
          findings.add(error(CONTENT_TYPE_MISMATCH, artifact.path, req.type))

      // Metadata schema (P-012, P-018, P-024)
      if typeDef.metadata_schema:
        if not artifact.metadata:
          findings.add(error(MISSING_METADATA, artifact.path, req.type))
        else if not jsonSchemaValidate(artifact.metadata, typeDef.metadata_schema):
          findings.add(error(SCHEMA_VIOLATION, artifact.path, req.type))

  // Step 8: Check unknown artifacts
  knownTypes = set(req.type for req in effectiveProfile.requirements)
  for artifact in pack.artifacts:
    if artifact.semantic_type and artifact.semantic_type not in knownTypes:
      findings.add(warning(UNKNOWN_ARTIFACT, artifact.path))

  return findings
```
