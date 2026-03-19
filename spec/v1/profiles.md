# Profiles & Semantic Validation

**Version:** 1.0
**Status:** Draft

> **Draft Specification:** This companion specification is under active development. The concepts and requirements defined here may change based on implementation feedback.

## 1. Introduction

A structurally valid Evidence Pack tells you nothing about whether it contains the *right* evidence. A pack could pass all integrity checks but be missing a required SOC 2 report, or include a pentest summary without the full findings.

Profiles solve this by declaring what a pack must contain.

### 1.1 Relationship to Core Spec

This spec builds on the reserved fields defined in [pack.md](pack.md) Section 10:

- `profiles` — Array of profile identifiers this pack claims to conform to
- `overlays` — Additional constraints applied on top of all profiles
- `profile_lock` — Pinned digests for reproducible validation

**Multiple profiles:** A pack may declare multiple profiles when it needs to satisfy independent compliance frameworks. Each profile is validated independently against the same artifact set. The pack passes only if ALL declared profiles pass.

**Core spec validators ignore these fields.** A Level 1 validator will accept a pack regardless of whether it satisfies its declared profile. Semantic validation requires a profile-aware validator implementing this companion spec.

### 1.2 Design Rationale

The core spec focuses on structural correctness and cryptographic integrity—stable, well-understood problems. Semantic validation is domain-specific and evolving. By keeping them separate, the core spec can remain stable while profiles iterate based on real-world usage.

### 1.3 Artifact Semantic Type Declaration

For semantic validation to work, artifacts must declare their semantic types. This companion spec defines a new optional artifact field:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `semantic_types` | Array | No | Type identifiers this artifact satisfies (e.g., `["evidencepack/soc2-report@v1"]`) |

An artifact matches a profile requirement when `requirement.type` appears in `artifact.semantic_types`.

**Multiple types:** A single artifact may satisfy multiple type requirements. For example, an AWS IAM configuration export could satisfy both `evidencepack/iam-summary@v1` and `evidencepack/cloud-config@v1`:

```json
{
  "type": "embedded",
  "path": "artifacts/aws-iam-config.json",
  "semantic_types": [
    "evidencepack/iam-summary@v1",
    "evidencepack/cloud-config@v1"
  ],
  "metadata": { ... }
}
```

**Interoperability:** The `semantic_types` field is defined by this companion spec, not the core spec. Adding `semantic_types` is backwards-compatible with validators that implement only the core spec, because they only need to parse and validate the fields defined by the core spec and can ignore additional JSON object members. Semantic validators implementing this spec MUST also accept unknown artifact fields they do not recognize (P-021).

**Naming rationale:** The field is named `semantic_types` (not `type_ids`) to clearly distinguish it from the core spec's `type` field ("embedded"/"reference") and to indicate that it carries semantic meaning for profile validation.

## 2. Core Concepts

| Concept | Description |
|---------|-------------|
| **Type** | A reusable definition for a category of evidence artifact (e.g., "SOC 2 Type II Report") |
| **Profile** | A named contract declaring what types a pack must include |
| **Overlay** | Additive constraints that modify a profile for specific contexts |

### 2.1 How They Compose

**Single profile:**
```json
{
  "spec_version": "1.0",
  "profiles": ["evidencepack/soc2-basic@v1"],
  "overlays": ["evidencepack/hipaa-overlay@v1"],
  "profile_lock": [
    { "id": "evidencepack/soc2-basic@v1", "digest": "sha256:abc123..." },
    { "id": "evidencepack/hipaa-overlay@v1", "digest": "sha256:def456..." },
    { "id": "evidencepack/soc2-report@v1", "digest": "sha256:789def..." }
  ],
  "artifacts": [...]
}
```

**Multiple profiles (independent frameworks):**
```json
{
  "spec_version": "1.0",
  "profiles": [
    "evidencepack/soc2-basic@v1",
    "acme/vendor-security@v1"
  ],
  "overlays": ["acme/enterprise-overlay@v1"],
  "profile_lock": [...],
  "artifacts": [...]
}
```

**Validation proceeds as follows:**

For each profile in `profiles`:
1. **Resolve** the profile, overlays, and all referenced types (verify digests if locked)
2. **Merge** overlays onto the profile to produce an effective profile
3. **Match** artifacts to requirements by checking if `requirement.type` is in `artifact.semantic_types`
4. **Validate** each artifact's metadata against its type's `metadata_schema`
5. **Check** cardinality, freshness, and metadata conditions
6. **Report** findings (pass, fail, or warning for each requirement)

**Independent validation:** Each profile is validated independently against the same artifact set. There is no merging between profiles. A pack passes only if ALL declared profiles pass (zero error-severity findings across all profiles). Overlays apply to ALL profiles.

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
GET https://schemas.evidencepack.org/types/evidencepack/soc2-report@v1
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
      "input_url": "https://schemas.evidencepack.org/examples/vcs-config/github-org/input.json",
      "output_url": "https://schemas.evidencepack.org/examples/vcs-config/github-org/output.json"
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

**Simple requirement (single type):**

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

**Complex requirement (control with flexible type matching):**

For compliance frameworks where a single control may be satisfied by different evidence types, or requires multiple types together, use `satisfied_by` with `any_of` or `all_of`:

```json
{
  "id": "acme/security-baseline@v1",
  "name": "ACME Security Baseline",
  "description": "Evidence requirements for ACME security compliance",
  "requirements": [
    {
      "control": "AC-01",
      "name": "Access Authorization",
      "satisfied_by": {
        "all_of": [
          { "type": "evidencepack/iam-summary@v1" },
          { "type": "evidencepack/idp-config@v1" }
        ]
      },
      "freshness": { "max_age_days": 90 },
      "metadata_conditions": {
        "all": [
          { "path": "mfa_enforced", "op": "eq", "value": true }
        ]
      }
    },
    {
      "control": "AU-01",
      "name": "Audit Logging",
      "satisfied_by": {
        "any_of": [
          { "type": "evidencepack/siem-config@v1" },
          { "type": "evidencepack/cloud-audit-logs@v1" }
        ]
      },
      "freshness": { "max_age_days": 90 }
    }
  ]
}
```

### 4.2 Requirement Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | String | — | Type identifier (use this OR `satisfied_by`, not both) |
| `satisfied_by` | Object | — | Flexible type matching with `any_of` or `all_of` |
| `control` | String | — | Compliance control identifier this requirement addresses |
| `name` | String | — | Human-readable name for this requirement |
| `required` | Boolean | `true` | Syntactic sugar for `cardinality.min` (see below) |
| `cardinality.min` | Integer | `1` if required, `0` otherwise | Minimum artifact count per type |
| `cardinality.max` | Integer | unlimited | Maximum artifact count per type |
| `freshness.max_age_days` | Integer | unlimited | Maximum days since `collected_at` |
| `metadata_conditions` | Object | — | Conditions on artifact metadata values (see Section 4.4) |

**`type` vs `satisfied_by`:** A requirement MUST specify either `type` (for simple single-type requirements) or `satisfied_by` (for complex requirements). Specifying both is an error.

**`satisfied_by` structure:**
- `any_of`: Array of type references. Requirement is satisfied if at least one type has matching artifacts.
- `all_of`: Array of type references. Requirement is satisfied only if ALL types have matching artifacts.

Each type reference in `any_of`/`all_of` is an object with a `type` field:
```json
{ "type": "evidencepack/iam-summary@v1" }
```

**`required` is syntactic sugar:** The `required` field only affects the default value of `cardinality.min` when `cardinality.min` is absent. If `cardinality.min` is explicitly set, it takes precedence. Validation uses the effective `minCount` to determine whether artifacts are missing:
- `minCount = cardinality.min` (if present), otherwise `1` if `required` is true, otherwise `0`
- Emit `MISSING_REQUIRED` when `minCount > 0` and artifact count is `0`
- Emit `CARDINALITY_VIOLATION` when artifact count is greater than `0` but less than `minCount`, or greater than `maxCount`

### 4.3 Requirement Type Uniqueness

Within a profile's `requirements` array, each `type` value MUST be unique (for simple requirements) and each `control` value MUST be unique (for control-based requirements). Duplicate entries produce ambiguity during validation and overlay merging.

Similarly, an overlay's `add_requirements` MUST NOT introduce a type or control that already exists in the effective profile at the time of application. If violated, validators MUST emit `DUPLICATE_REQUIREMENT_TYPE` as an error.

### 4.4 Metadata Conditions

Metadata conditions allow profiles to validate specific values in artifact metadata, not just presence. This is essential for compliance where the *value* matters (e.g., MFA must be enabled, not just reported).

**Structure:**

```json
{
  "metadata_conditions": {
    "all": [
      { "path": "mfa_enforced", "op": "eq", "value": true },
      { "path": "unused_credentials_exist", "op": "eq", "value": false }
    ]
  }
}
```

**Condition fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | String | Yes | JSON path to the metadata field (dot notation for nested) |
| `op` | String | Yes | Comparison operator |
| `value` | Any | Depends | Expected value (required for most operators) |

**Operators:**

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equals | `{ "path": "mfa_enforced", "op": "eq", "value": true }` |
| `neq` | Not equals | `{ "path": "status", "op": "neq", "value": "disabled" }` |
| `gt` | Greater than | `{ "path": "password_min_length", "op": "gt", "value": 12 }` |
| `gte` | Greater than or equal | `{ "path": "retention_days", "op": "gte", "value": 90 }` |
| `lt` | Less than | `{ "path": "failed_checks", "op": "lt", "value": 5 }` |
| `lte` | Less than or equal | `{ "path": "critical_findings", "op": "lte", "value": 0 }` |
| `in` | Value in array | `{ "path": "provider", "op": "in", "value": ["aws", "gcp", "azure"] }` |
| `contains` | Array contains value | `{ "path": "services", "op": "contains", "value": "iam" }` |
| `contains_all` | Array contains all values | `{ "path": "regions", "op": "contains_all", "value": ["us-east-1", "us-west-2"] }` |
| `exists` | Field exists and is not null | `{ "path": "audit_log_destination", "op": "exists" }` |
| `regex` | Matches regular expression | `{ "path": "policy_version", "op": "regex", "value": "^v[0-9]+\\.[0-9]+$" }` |

**Nested paths:** Use dot notation to access nested fields:
```json
{ "path": "encryption.at_rest_enabled", "op": "eq", "value": true }
```

**Array indexing:** Use bracket notation for array access:
```json
{ "path": "findings[0].severity", "op": "neq", "value": "critical" }
```

**Condition groups:** Use `all` (AND) or `any` (OR) to group conditions:
```json
{
  "metadata_conditions": {
    "any": [
      { "path": "mfa_enforced", "op": "eq", "value": true },
      { "path": "sso_enabled", "op": "eq", "value": true }
    ]
  }
}
```

**Validation behavior:**
- If `metadata_conditions` is present and any condition fails, emit `METADATA_CONDITION_FAILED`
- If a path does not exist in the metadata, the condition fails (unless using `exists` with expected `false`)
- Conditions are evaluated against each matching artifact independently

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

An artifact matches a type when:

```
requirement.type in artifact.semantic_types
```

For requirements using `satisfied_by`:
- **`any_of`**: Requirement is satisfied if at least one type in the array has matching artifacts meeting cardinality
- **`all_of`**: Requirement is satisfied if ALL types in the array have matching artifacts meeting cardinality

Artifacts without a `semantic_types` field (or with an empty array) are not matched to any requirement and produce an `UNTYPED_ARTIFACT` warning.

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

2. **Index artifacts** — Group artifacts by each type in their `semantic_types` array. An artifact with multiple semantic types appears in multiple groups. Emit `UNTYPED_ARTIFACT` warning for artifacts without `semantic_types` or with an empty array. Emit `INVALID_SEMANTIC_TYPE` warning for any entry in `semantic_types` that does not match the identifier format (Section 3.3).

3. **Validate each profile independently** — For each profile in `profiles`:

   a. **Resolve profile** — Resolve per Section 7 and verify locks per P-002. Emit `PROFILE_NOT_FOUND` or `DIGEST_MISMATCH` as appropriate.

   b. **Resolve and merge overlays** — If `overlays` is absent, treat it as an empty list. For each overlay in declared order, resolve and verify locks. Apply merge semantics per Section 5.2. Emit `OVERLAY_NOT_FOUND`, `DIGEST_MISMATCH`, or `OVERLAY_TARGET_NOT_FOUND` as appropriate. Note: Overlays apply to ALL profiles.

   c. **Resolve types** — For each type referenced by the effective profile, resolve and verify locks. Emit `TYPE_NOT_FOUND` or `DIGEST_MISMATCH` as appropriate.

   d. **Check unpinned dependencies** — For the profile, all overlays, and all types in the effective profile, emit `UNPINNED_DEPENDENCY` warning if not in `profile_lock`. Validators SHOULD emit at most one `UNPINNED_DEPENDENCY` per identifier per validation run.

   e. **Check requirements** — For each requirement in the effective profile:
      - **Simple requirements** (using `type`): Check cardinality, freshness, content_type, metadata_schema, and metadata_conditions against matching artifacts
      - **Complex requirements** (using `satisfied_by`):
        - For `any_of`: Check if at least one type has artifacts meeting cardinality. If so, apply freshness and metadata_conditions to those artifacts.
        - For `all_of`: Check that ALL types have artifacts meeting cardinality. Apply freshness and metadata_conditions to all matching artifacts.
      - If type unavailable, skip checks that depend on the type definition (`content_type`, `metadata_schema`) per P-022. Cardinality, freshness, and metadata_conditions still run.
      - For each matching artifact: check freshness (P-010/P-011/P-027), `content_type` (P-019), `metadata` (P-012/P-018/P-024), and `metadata_conditions` (Section 4.4)

   f. **Check unknown artifacts** — Emit `UNKNOWN_ARTIFACT` warning for artifacts with `semantic_types` where none of the types match any requirement in this profile.

4. **Aggregate results** — The pack passes only if ALL profiles pass (zero error-severity findings across all profiles). Findings from each profile include the `profile` field to indicate which profile produced them.

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
      "profile": "evidencepack/soc2-basic@v1",
      "type": "evidencepack/soc2-report@v1"
    },
    {
      "severity": "error",
      "code": "STALE_ARTIFACT",
      "message": "Artifact is 400 days old, max allowed is 365",
      "profile": "evidencepack/soc2-basic@v1",
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
  },
  "by_profile": {
    "evidencepack/soc2-basic@v1": { "errors": 2, "warnings": 0, "passed": false },
    "acme/vendor-security@v1": { "errors": 0, "warnings": 1, "passed": true }
  }
}
```

**Pass/fail semantics:** Validation passes if and only if there are zero error-severity findings across ALL profiles. Any error finding (including early errors like `DUPLICATE_LOCK_ENTRY` or `PROFILE_NOT_FOUND`) means `summary.passed = false`. Validators MAY return early when a profile cannot be resolved or fails lock verification, because subsequent steps depend on it. Otherwise, validators SHOULD continue collecting findings after encountering errors to provide complete diagnostics.

**Finding context fields:** Findings MUST include `profile` for any finding scoped to a specific profile. Findings SHOULD include `type` for requirement-scoped issues (e.g., `MISSING_REQUIRED`, `CARDINALITY_VIOLATION`) and `path` for artifact-specific issues (e.g., `STALE_ARTIFACT`, `SCHEMA_VIOLATION`). Artifact-level findings SHOULD include `profile`, `path`, and `type` to provide complete context for diagnostics. Profile-independent findings (e.g., `DUPLICATE_LOCK_ENTRY`, `UNTYPED_ARTIFACT`) omit the `profile` field.

**Per-profile summary:** When validating multiple profiles, the `by_profile` object provides a breakdown of findings per profile. This allows consumers to see which specific profiles passed or failed.

**Findings ordering:** Validators SHOULD emit findings in a stable order to facilitate testing and debugging. Recommended order: lock errors, untyped artifact warnings, then for each profile in declared order: resolution errors (profile, overlays, types), overlay merge errors, unpinned dependency warnings, requirement checks in profile order, artifact checks in artifact index order, unknown artifact warnings.

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
| `METADATA_CONDITION_FAILED` | error | Artifact metadata fails a `metadata_conditions` check |
| `REQUIREMENT_NOT_SATISFIED` | error | Complex requirement (`any_of`/`all_of`) not satisfied |
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
| `UNTYPED_ARTIFACT` | warning | Artifact has no `semantic_types` field or empty array |
| `INVALID_SEMANTIC_TYPE` | warning | Entry in `semantic_types` does not match the identifier format |
| `UNKNOWN_ARTIFACT` | warning | Artifact's `semantic_types` don't match any profile requirement |

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

**Default registry:** The default public registry for open types and profiles is:

```
https://schemas.evidencepack.org
```

This registry hosts the `evidencepack/*` namespace and is the canonical source for reference types and profiles defined in this spec.

**Registry configuration:**

```yaml
# Example validator configuration
registries:
  - url: https://schemas.evidencepack.org
    trusted: true

  # Private registry with bearer token
  - url: https://registry.acme.com
    trusted: true
    auth:
      type: bearer
      token_env: ACME_REGISTRY_TOKEN
    secrets:
      - ACME_REGISTRY_TOKEN

  # Private registry with basic auth
  - url: https://internal.example.com/profiles
    trusted: true
    auth:
      type: basic
      username_env: REGISTRY_USER
      password_env: REGISTRY_PASSWORD
    secrets:
      - REGISTRY_USER
      - REGISTRY_PASSWORD

  # OAuth2 client credentials
  - url: https://enterprise.example.com
    trusted: true
    auth:
      type: oauth2
      token_url: https://auth.example.com/oauth/token
      client_id_env: OAUTH_CLIENT_ID
      client_secret_env: OAUTH_CLIENT_SECRET
      scope: profiles:read
    secrets:
      - OAUTH_CLIENT_ID
      - OAUTH_CLIENT_SECRET

offline_definitions:
  - path: /etc/evidence-pack/types/
```

**Authentication methods:**

| Method | Description | Use Case |
|--------|-------------|----------|
| `bearer` | Static bearer token sent as `Authorization: Bearer <token>` | API keys, personal access tokens |
| `basic` | HTTP Basic auth sent as `Authorization: Basic <base64>` | Simple username/password systems |
| `oauth2` | OAuth2 client credentials flow | Enterprise SSO, machine-to-machine |

**Secrets allowlist:** Registry auth follows the same security model as epack tools and collectors. The `secrets` array explicitly lists which environment variables may be accessed for authentication. Validators MUST only read env vars listed in `secrets`. Auth fields use `*_env` suffix (e.g., `token_env`, `password_env`) to reference env var names, not values.

**Reserved prefixes:** Validators MUST reject secret names with reserved prefixes: `EPACK_*`, `LD_*`, `DYLD_*`, `_*`. This prevents leaking protocol variables or hijacking the dynamic linker.

**Token caching:** For `oauth2` auth, validators SHOULD cache access tokens and refresh them when expired. The token endpoint response MUST include `expires_in` for the validator to manage expiry.

**Secret handling:** Validators MUST NOT log, persist, or include secrets in error messages. Auth credentials are read at runtime from the allowlisted env vars and used only for HTTP requests to the configured registry.

**Namespace ownership:** Profile and type identifiers use namespaces (the part before `/`) to indicate ownership:
- `evidencepack/*` — Open reference definitions hosted at schemas.evidencepack.org
- `acme/*` — Organization-owned definitions (hosted at organization's registry)

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

- All declared profiles
- All declared overlays
- All types referenced by all effective profiles (after overlay merging)

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
| `untyped-artifact.zip` | Pack with artifact missing `semantic_types` field | UNTYPED_ARTIFACT (warning) |

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
| P-009 | MUST | Artifact type matching MUST check if `requirement.type` is in `artifact.semantic_types` |
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
| P-033 | MUST | Requirements MUST specify either `type` OR `satisfied_by`, not both |
| P-034 | MUST | For `all_of` requirements, ALL referenced types MUST have matching artifacts |
| P-035 | MUST | For `any_of` requirements, at least one referenced type MUST have matching artifacts |
| P-036 | MUST | If `metadata_conditions` is present and any condition fails, emit `METADATA_CONDITION_FAILED` |
| P-037 | MUST | Metadata condition paths MUST use dot notation for nested fields |
| P-038 | SHOULD | Profiles for compliance frameworks SHOULD include `control` identifiers |
| P-039 | MUST | Each profile in `profiles` MUST be validated independently against the same artifact set |
| P-040 | MUST | Validation MUST pass only if ALL declared profiles pass (zero errors across all profiles) |
| P-041 | MUST | Overlays MUST apply to ALL profiles when multiple profiles are declared |
| P-042 | MUST | Profile-scoped findings MUST include the `profile` field indicating which profile produced them |
| P-043 | MUST | Validators MUST only read env vars explicitly listed in the registry `secrets` array |
| P-044 | MUST | Validators MUST reject secret names with reserved prefixes: `EPACK_*`, `LD_*`, `DYLD_*`, `_*` |
| P-045 | MUST NOT | Validators MUST NOT log, persist, or include secrets in error messages |

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

// Helper: add finding with profile context
function addFinding(findings, finding, profileId):
  if profileId:
    finding.profile = profileId
  findings.add(finding)

function validateSemantic(pack):
  findings = []
  lockedDigests = {}  // id -> digest from profile_lock
  byProfile = {}  // profileId -> { errors, warnings }

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

  // Step 2: Index artifacts by semantic_types (shared across all profiles)
  artifactsByType = {}
  for artifact in pack.artifacts:
    if artifact.semantic_types and len(artifact.semantic_types) > 0:
      for semType in artifact.semantic_types:
        artifactsByType[semType].append(artifact)
    else:
      findings.add(warning(UNTYPED_ARTIFACT, artifact.path))

  // Step 3: Validate each profile independently (P-039, P-040)
  profiles = pack.profiles or []
  overlays = pack.overlays or []  // overlays apply to ALL profiles

  for profileId in profiles:
    byProfile[profileId] = { errors: 0, warnings: 0 }

    // 3a: Resolve profile (P-001, P-002, P-004)
    result = resolveLocked(profileId, lockedDigests, PROFILE_NOT_FOUND)
    if not result.ok:
      addFinding(findings, result.err, profileId)
      byProfile[profileId].errors += 1
      continue  // skip this profile, continue with next
    profileWrapper = result.wrapper

    // 3b: Resolve and merge overlays (P-002, P-005, P-013)
    effectiveProfile = copy(profileWrapper.definition)
    for overlayId in overlays:
      result = resolveLocked(overlayId, lockedDigests, OVERLAY_NOT_FOUND)
      if not result.ok:
        addFinding(findings, result.err, profileId)
        byProfile[profileId].errors += 1
        continue
      effectiveProfile = merge(effectiveProfile, result.wrapper.definition, findings, profileId)

    // 3c: Resolve types (P-002, P-004)
    typeWrappers = {}  // type id -> { id, digest, definition }
    for req in effectiveProfile.requirements:
      typeIds = getTypeIds(req)  // handles both simple type and satisfied_by
      for typeId in typeIds:
        if typeId not in typeWrappers:
          result = resolveLocked(typeId, lockedDigests, TYPE_NOT_FOUND)
          if not result.ok:
            addFinding(findings, result.err, profileId)
            byProfile[profileId].errors += 1
            continue
          typeWrappers[typeId] = result.wrapper

    // 3d: Check unpinned dependencies (P-015, P-016)
    warnedUnpinned = set()
    allDeps = [profileId] + overlays + [t for t in typeWrappers.keys()]
    for dep in allDeps:
      if dep not in lockedDigests and dep not in warnedUnpinned:
        addFinding(findings, warning(UNPINNED_DEPENDENCY, dep), profileId)
        byProfile[profileId].warnings += 1
        warnedUnpinned.add(dep)

    // 3e: Check requirements (P-009, P-018, P-019, P-022, P-033-P-036)
    for req in effectiveProfile.requirements:
      // Determine matching artifacts based on simple type or satisfied_by
      minCount = req.cardinality?.min ?? (1 if req.required else 0)
      maxCount = req.cardinality?.max or infinity

      if req.type:
        matchingArtifacts = artifactsByType.get(req.type, [])
        typeWrapper = typeWrappers.get(req.type)
      else if req.satisfied_by:
        // Handle any_of / all_of
        if req.satisfied_by.any_of:
          matchingArtifacts = []
          typeWrapper = null  // Multiple types, skip type-specific checks
          satisfied = false
          for typeRef in req.satisfied_by.any_of:
            typeArtifacts = artifactsByType.get(typeRef.type, [])
            if len(typeArtifacts) >= minCount:
              matchingArtifacts.extend(typeArtifacts)
              satisfied = true
          if not satisfied and minCount > 0:
            addFinding(findings, error(REQUIREMENT_NOT_SATISFIED, req.control or req.name), profileId)
            byProfile[profileId].errors += 1
            continue
        else if req.satisfied_by.all_of:
          matchingArtifacts = []
          typeWrapper = null
          allSatisfied = true
          for typeRef in req.satisfied_by.all_of:
            typeArtifacts = artifactsByType.get(typeRef.type, [])
            if len(typeArtifacts) < minCount:
              allSatisfied = false
              break
            matchingArtifacts.extend(typeArtifacts)
          if not allSatisfied and minCount > 0:
            addFinding(findings, error(REQUIREMENT_NOT_SATISFIED, req.control or req.name), profileId)
            byProfile[profileId].errors += 1
            continue
      else:
        matchingArtifacts = []
        typeWrapper = null

      // Cardinality (always checked, even if type unavailable)
      count = len(matchingArtifacts)
      if minCount > 0 and count == 0:
        addFinding(findings, error(MISSING_REQUIRED, req.type), profileId)
        byProfile[profileId].errors += 1
      else if count < minCount:
        addFinding(findings, error(CARDINALITY_VIOLATION, req.type), profileId)
        byProfile[profileId].errors += 1
      if count > maxCount:
        addFinding(findings, error(CARDINALITY_VIOLATION, req.type), profileId)
        byProfile[profileId].errors += 1

      // Freshness (always checked, even if type unavailable) (P-010, P-011, P-027)
      if req.freshness?.max_age_days:
        for artifact in matchingArtifacts:
          if not artifact.collected_at:
            addFinding(findings, error(MISSING_COLLECTED_AT, artifact.path, req.type), profileId)
            byProfile[profileId].errors += 1
          else if not isValidTimestamp(artifact.collected_at):
            addFinding(findings, error(INVALID_COLLECTED_AT, artifact.path, req.type), profileId)
            byProfile[profileId].errors += 1
          else:
            age = daysSince(artifact.collected_at)
            if age > req.freshness.max_age_days:
              addFinding(findings, error(STALE_ARTIFACT, artifact.path, req.type), profileId)
              byProfile[profileId].errors += 1

      // Skip type-based checks if type unavailable (P-022)
      if not typeWrapper:
        continue

      typeDef = typeWrapper.definition
      for artifact in matchingArtifacts:
        // Content type (P-019)
        if typeDef.content_types:
          if not artifact.content_type:
            addFinding(findings, error(MISSING_CONTENT_TYPE, artifact.path, req.type), profileId)
            byProfile[profileId].errors += 1
          else if artifact.content_type not in typeDef.content_types:
            addFinding(findings, error(CONTENT_TYPE_MISMATCH, artifact.path, req.type), profileId)
            byProfile[profileId].errors += 1

        // Metadata schema (P-012, P-018, P-024)
        if typeDef.metadata_schema:
          if not artifact.metadata:
            addFinding(findings, error(MISSING_METADATA, artifact.path, req.type), profileId)
            byProfile[profileId].errors += 1
          else if not jsonSchemaValidate(artifact.metadata, typeDef.metadata_schema):
            addFinding(findings, error(SCHEMA_VIOLATION, artifact.path, req.type), profileId)
            byProfile[profileId].errors += 1

      // Metadata conditions (P-036, P-037) - checked for all matching artifacts
      if req.metadata_conditions:
        for artifact in matchingArtifacts:
          if not evaluateConditions(artifact.metadata, req.metadata_conditions):
            addFinding(findings, error(METADATA_CONDITION_FAILED, artifact.path, req.control or req.type), profileId)
            byProfile[profileId].errors += 1

    // 3f: Check unknown artifacts for this profile
    knownTypes = set()
    for req in effectiveProfile.requirements:
      if req.type:
        knownTypes.add(req.type)
      else if req.satisfied_by:
        for typeRef in (req.satisfied_by.any_of or req.satisfied_by.all_of or []):
          knownTypes.add(typeRef.type)
    for artifact in pack.artifacts:
      if artifact.semantic_types:
        if not any(t in knownTypes for t in artifact.semantic_types):
          addFinding(findings, warning(UNKNOWN_ARTIFACT, artifact.path), profileId)
          byProfile[profileId].warnings += 1

    // Compute per-profile passed status
    byProfile[profileId].passed = (byProfile[profileId].errors == 0)

  // Step 4: Aggregate results
  totalErrors = sum(p.errors for p in byProfile.values())
  totalWarnings = sum(p.warnings for p in byProfile.values())
  // Include profile-independent errors (e.g., DUPLICATE_LOCK_ENTRY)
  totalErrors += count(f for f in findings if f.severity == "error" and not f.profile)

  return {
    findings: findings,
    summary: { errors: totalErrors, warnings: totalWarnings, passed: totalErrors == 0 },
    by_profile: byProfile
  }
```
