# Profiles & Semantic Validation

**Version:** 1.0
**Status:** Draft

> **Draft Specification:** This companion specification is under active development. The concepts and requirements defined here may change based on implementation feedback.

## 1. Introduction

A structurally valid Evidence Pack tells you nothing about whether it contains the *right* evidence. A pack could pass all integrity checks but be missing a required SOC 2 report, or carry identity provider posture showing that MFA is not enforced.

Profiles solve this by stating the controls a pack is checked against. Each requirement in a profile is a control. Its name is the statement a reviewer reads, and its clauses are the tests that check it against the pack's artifacts.

### 1.1 Relationship to Core Spec

This spec builds on two parts of [pack.md](pack.md):

- The artifact `schema` field (Section 4.2.1), which names the type of an artifact's content. Profiles match artifacts by this field.
- The reserved `profiles` and `overlays` manifest fields (Section 10), which record the profile and overlays a pack was built to be validated against.

**Core spec validators ignore profiles.** A Level 1 validator accepts a pack regardless of whether it satisfies a profile. Semantic validation requires a validator that implements this companion spec.

**One profile per run.** A validator run evaluates one profile, after applying its overlays in order. To check a pack against several frameworks, run the validator once for each profile.

### 1.2 Design Rationale

The core spec covers structural correctness and cryptographic integrity, which are stable and well understood. Semantic validation is domain-specific and still evolving. Keeping the two apart lets the core spec stay stable while profiles change with real-world use.

### 1.3 Artifact Types

A profile refers to evidence by type. A type is the identifier in an artifact's manifest `schema` field, such as `evidencepack/idp-posture@v1`. An artifact matches a clause when its `schema` equals the clause's `type`, byte for byte. Artifacts without a `schema` never match.

The content of an artifact with a `schema` MUST be a JSON document. Conditions read values from that document with JSONPath (Section 4.5). Two kinds of types are in use.

- **Posture types**, such as `evidencepack/cloud-posture@v1`. The artifact is the normalized posture document, and its content follows the type's schema, such as [schemas/idp-posture.md](schemas/idp-posture.md).
- **Document types**, such as `evidencepack/pentest-report@v1`. A document's file, such as a PDF, carries no `schema`. Beside it the pack holds a JSON metadata record whose `schema` is the document's type. The record points to the file and holds the fields read from the document under `metadata`.

A metadata record for a penetration test report:

```json
{
  "document_id": "doc-3a7d9e21",
  "name": "2026 Penetration Test",
  "filename": "pentest-2026.pdf",
  "file_path": "artifacts/documents/doc-3a7d9e21/pentest-2026.pdf",
  "text_path": "artifacts/documents/doc-3a7d9e21/text.json",
  "metadata": {
    "test_date": "2026-05-12",
    "assessor": "Example Security LLC",
    "findings_summary": { "critical": 0, "high": 1 }
  }
}
```

Its entry in the manifest artifact index:

```json
{
  "type": "embedded",
  "path": "artifacts/documents/doc-3a7d9e21/metadata.json",
  "digest": "sha256:5d41402abc4b2a76b9719d911017c592ae2e7f6a1f2b3c4d5e6f708192a3b4c5",
  "size": 412,
  "content_type": "application/json",
  "schema": "evidencepack/pentest-report@v1",
  "collected_at": "2026-10-28T09:00:00Z"
}
```

A condition on this document reads `$.metadata.test_date`.

### 1.4 Control Mapping Artifacts

Some requirements are answered by evidence that no machine check reads, such as a written policy, a monitoring export, or a signed agreement. A control mapping records which evidence in the pack the publisher offers for which requirements.

A **control mapping** is an ordinary embedded artifact, identified by `schema: "evidencepack/control-mapping@v1"` in the manifest artifact index. Its entries link evidence in the pack to requirements of a profile at three granularities: a whole artifact, a value inside a JSON artifact (RFC 6901 pointer), or a document citation (a quoted passage or a table row). The format is defined in [schemas/control-mapping.md](schemas/control-mapping.md).

Control mappings are publisher assertions. Like the core spec's `sources` field, they are sealed and signed with the pack and have no effect on validation outcomes. Validators that support them report entries as *asserted support* beside requirement outcomes (Section 6.6). Validators without mapping support see only an ordinary artifact. Entries may reference any requirement, so a machine-checked requirement shows its computed outcome and its asserted support side by side. A requirement declares `manual: true` (Section 4.3) when no machine check answers it at all, and it is answered by asserted support and human review.

## 2. Core Concepts

| Concept | Description |
|---------|-------------|
| **Type** | The identifier of an artifact's content schema, carried in its manifest `schema` field (Section 3) |
| **Profile** | A named set of requirements a pack is checked against (Section 4) |
| **Overlay** | Changes to a profile for one pipeline or audience, which replace, skip, or add requirements (Section 5) |
| **Control mapping** | Publisher-authored assertions linking evidence in a pack to a profile's requirements (Section 1.4) |

### 2.1 How They Compose

A validator run takes one profile and an ordered list of overlays, usually from the validator's configuration. It then:

1. **Loads** the profile and overlays, and checks the profile's digest against the pack's manifest when the manifest records one (Section 7)
2. **Merges** the overlays into the profile, in order, to produce the effective profile (Section 5)
3. **Indexes** the pack's artifacts by `schema` (Section 6.1)
4. **Evaluates** each requirement of the effective profile (Sections 6.2 to 6.5)
5. **Attaches** asserted support from control mappings (Section 6.6)
6. **Reports** each requirement's outcome and the pack's overall status (Section 6.7)

A pack built for a profile records it, with its overlays, in the manifest:

```json
{
  "spec_version": "1.0",
  "profiles": [
    { "source": "evidencepack/soc2-basic@v1", "digest": "sha256:2c46055d3ad6c8e0b1f4a7d29e5c3b8f6a0d1e4c7b2f9a5e8d3c6b1f0a7e4d29" }
  ],
  "overlays": [
    { "source": "profiles/overlays/hipaa.json", "digest": "sha256:9b1e70c4a2f8d6e3b5c7a9f1d0e2c4b6a8f0e1d3c5b7a9f2e4d6c8b0a1f3e5d7" }
  ],
  "artifacts": []
}
```

## 3. Types

A type names what an artifact's content is. A posture type's content follows its schema in [schemas/](schemas/). A document type's metadata record carries the document's fields under `metadata`, described by the type's `metadata_schema`.

### 3.1 Type Definition

A type definition documents a type. Validators do not need type definitions to evaluate a profile, since clauses match on the identifier alone. Authoring tools use them to describe types and to check condition paths (Section 4.7).

```json
{
  "id": "evidencepack/pentest-report@v1",
  "name": "Penetration Test Report",
  "description": "Summary or full report from a penetration test engagement conducted by a qualified security assessor.",
  "content_types": ["application/pdf", "application/json"],
  "metadata_schema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
      "test_date": { "type": "string", "format": "date" },
      "assessor": { "type": "string" }
    },
    "required": ["test_date", "assessor"]
  }
}
```

### 3.2 Type Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Identifier in `namespace/name@vN` format (Section 3.3) |
| `name` | String | Yes | Human-readable name |
| `description` | String | No | What the type represents. A posture type names its schema page |
| `content_types` | Array | No | MIME types of the file a document type describes |
| `metadata_schema` | Object | No | JSON Schema for the `metadata` object of a document type's metadata record |
| `controls` | Array | No | Control framework references this type addresses |
| `examples` | Array | No | Conformance examples for collector testing (Section 3.4) |

Validators are not required to check an artifact's content against its type's schema. A value that does not fit the schema shows up through the conditions that read it.

### 3.3 Identifier Format

Type identifiers, and the identifiers of profiles and overlays published for reuse, MUST follow this format:

```
Format: namespace/name@vN
```

Where:
- `namespace`: lowercase ASCII alphanumeric, hyphens, underscores, 1 to 63 characters, no leading or trailing hyphen
- `name`: lowercase ASCII alphanumeric, hyphens, underscores, 1 to 63 characters, no leading or trailing hyphen
- `vN`: literal `v` followed by a positive integer with no leading zeros

**Regex:** `^[a-z0-9][a-z0-9_-]{0,62}/[a-z0-9][a-z0-9_-]{0,62}@v[1-9][0-9]*$`

**Examples:**
- Valid: `evidencepack/soc2-report@v1`, `acme/internal-policy@v2`, `my_org/custom_type@v10`
- Invalid: `SOC2-Report@v1` (uppercase), `acme/policy@latest` (non-numeric), `acme/policy@v01` (leading zero)

A profile kept inside one project MAY use any non-empty `id`.

### 3.4 Type Examples

Types MAY include an `examples` array for collector conformance testing. Each example provides a sample input (a mocked API response) and the expected output artifact, so a collector's output can be checked automatically.

**Example structure:**

```json
{
  "id": "evidencepack/vcs-posture@v1",
  "name": "Version Control Posture",
  "examples": [
    {
      "name": "github-org",
      "description": "GitHub organization with mixed branch protection",
      "input_url": "https://schemas.evidencepack.org/examples/vcs-posture/github-org/input.json",
      "output_url": "https://schemas.evidencepack.org/examples/vcs-posture/github-org/output.json"
    }
  ]
}
```

**Example fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | String | Yes | Short identifier for this example (lowercase, alphanumeric, hyphens) |
| `description` | String | No | The scenario this example represents |
| `input_url` | String | Yes | URL of the sample API response (collector input) |
| `output_url` | String | Yes | URL of the expected artifact |

**Conformance testing:** A collector is conformant for a type when its output matches the expected output for every example. The procedure is:

1. Fetch the example input from `input_url`
2. Run the collector with the input
3. Check the collector's output against the type's schema
4. Compare the collector's output to the expected output from `output_url`

## 4. Profiles

A profile is a named set of requirements. A machine-checked requirement lists clauses that read artifacts of a type. A manual requirement is answered by people, with control mappings pointing at the evidence.

Profiles and overlays are JSON or YAML documents. The examples in this spec use JSON. In YAML, quote `version` so a value like `1.0` stays a string.

### 4.1 Profile Structure

A reference profile with two machine-checked requirements and one manual requirement:

```json
{
  "id": "evidencepack/soc2-basic@v1",
  "name": "SOC 2 Basic",
  "version": "1.0.0",
  "description": "Minimum evidence for a basic SOC 2 security review.",
  "requirements": [
    {
      "id": "soc2-report",
      "name": "A SOC 2 Type II report covers the last 12 months",
      "category": "Attestation",
      "satisfied_by": {
        "any_of": [
          {
            "type": "evidencepack/soc2-report@v1",
            "metadata_conditions": {
              "all": [
                { "path": "$.metadata.report_type", "op": "eq", "value": "type_ii" },
                { "path": "$.metadata.report_period_end", "op": "max_age_days", "value": 365 }
              ]
            }
          }
        ]
      }
    },
    {
      "id": "pentest",
      "name": "A penetration test was performed in the last 12 months",
      "category": "Security Testing",
      "satisfied_by": {
        "any_of": [
          {
            "type": "evidencepack/pentest-report@v1",
            "metadata_conditions": {
              "all": [
                { "path": "$.metadata.test_date", "op": "max_age_days", "value": 365 }
              ]
            }
          }
        ]
      }
    },
    {
      "id": "security-policy",
      "name": "An information security policy is in place",
      "category": "Policy",
      "manual": true,
      "description": "Current version, approved by management and reviewed at least annually"
    }
  ]
}
```

A framework profile that combines evidence types, keys requirements to controls, and leaves some to human review:

```json
{
  "id": "acme/security-baseline@v1",
  "name": "ACME Security Baseline",
  "version": "1.0.0",
  "requirements": [
    {
      "id": "access-authorization",
      "control": "AC-01",
      "name": "Sign-in requires strong authentication",
      "category": "Access Control",
      "satisfied_by": {
        "all_of": [
          {
            "type": "evidencepack/idp-posture@v1",
            "freshness": { "max_age_days": 90 },
            "metadata_conditions": {
              "all": [
                { "path": "$.policy.phishing_resistant_required_for_privileged", "op": "eq", "value": true }
              ]
            }
          },
          {
            "type": "evidencepack/access-posture@v1",
            "artifacts": "all",
            "freshness": { "max_age_days": 90 },
            "metadata_conditions": {
              "all": [
                { "path": "$.sign_in.mfa_required", "op": "eq", "value": true }
              ]
            }
          }
        ]
      }
    },
    {
      "id": "threat-detection",
      "control": "SI-04",
      "name": "Threats are detected on endpoints or in the cloud",
      "category": "Monitoring",
      "satisfied_by": {
        "any_of": [
          {
            "type": "evidencepack/endpoint-posture@v1",
            "metadata_conditions": {
              "all": [
                { "path": "$.protection.edr_installed_pct", "op": "gte", "value": 95 }
              ]
            }
          },
          {
            "type": "evidencepack/cloud-posture@v1",
            "metadata_conditions": {
              "all": [
                { "path": "$.accounts[*].threat_detection.enabled", "op": "eq", "value": true, "cardinality": "all" }
              ]
            }
          }
        ]
      }
    },
    {
      "id": "audit-logging",
      "control": "AU-01",
      "name": "Production audit logs are monitored",
      "category": "Monitoring",
      "manual": true,
      "description": "Export of the alert rules that watch production audit logs"
    },
    {
      "id": "security-policy",
      "control": "PL-01",
      "name": "An information security policy is approved and reviewed annually",
      "category": "Policy",
      "manual": true
    },
    {
      "id": "access-review",
      "control": "AC-02",
      "name": "User access is reviewed every quarter",
      "category": "Access Control",
      "manual": true
    }
  ]
}
```

### 4.2 Profile Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | Yes | Stable identifier. A profile published for reuse follows Section 3.3 |
| `name` | String | Yes | Human-readable name |
| `version` | String | Yes | Profile version, such as `1.0.0` |
| `description` | String | No | What the profile covers |
| `requirements` | Array | Yes | One or more requirements (Section 4.3) |

### 4.3 Requirements

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | See below | Stable identifier, unique within the effective profile |
| `name` | String | Yes | The control statement a reviewer reads, such as "MFA is enforced for every user" |
| `control` | String | No | Framework control this requirement answers, such as `CC6.1` |
| `description` | String | No | Prose stating what satisfies the requirement (see below) |
| `category` | String | No | Grouping used in result summaries |
| `satisfied_by` | Object | See below | Exactly one of `any_of` or `all_of`, each a non-empty array of clauses (Section 4.4) |
| `manual` | Boolean | See below | `true` when no machine check answers the requirement |

A requirement is either machine-checked or manual.

- A **machine-checked requirement** has `id` and `satisfied_by`, and `manual` is absent or `false`.
- A **manual requirement** has `manual: true` and no `satisfied_by`, and carries `id` or `control` so mapping entries can reference it. It is answered by asserted support (Section 1.4) and human review, and it is never scored (Section 6.5).

**`any_of` and `all_of`:** With `any_of`, the requirement passes when any clause passes. With `all_of`, it passes only when every clause passes. Section 6.4 defines how unknown and graded clauses combine.

**Requirement identifiers:** `id` names a requirement independently of its control or its clauses. Overlays find requirements by `id`, results report it, and it is the preferred join key for control mapping entries. Authors SHOULD keep `id` values stable across profile versions. `id` values MUST be unique within the effective profile.

**Requirement descriptions:** `description` is optional prose stating what would satisfy the requirement, written for the person fulfilling or reviewing it. For a requested document it might read "Current version, stating a review cadence of at least annually and naming an approving officer". It carries expectations a machine check cannot express and is most useful on manual requirements. Validators SHOULD echo it on the requirement's result so reviewers see the criteria beside the outcome and any asserted support. It never affects evaluation.

### 4.4 Clauses

A clause tests the artifacts of one type.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | String | Yes | Type to read, matched against artifact `schema` |
| `severity` | String | No | `critical`, `high`, `medium`, or `low`. Marks a clause that describes a graded failure (Section 4.6) |
| `artifacts` | String | No | `any`, the default, or `all`. Whether one artifact of the type or every one must satisfy the clause |
| `freshness.max_age_days` | Integer | No | The artifact must have been collected within this many days, by its `collected_at`. At least 1 |
| `metadata_conditions.all` | Array | No | Conditions that must all hold (Section 4.5) |

A clause passes when one artifact of its type is fresh enough and satisfies every condition. A clause with no conditions passes when such an artifact exists. Section 6.2 defines the evaluation in full.

**Every artifact of a type:** A pack can hold several artifacts of one type, such as `idp-posture` from two identity providers, or one `access-posture` artifact for each system. With `artifacts: all`, the clause passes only when every artifact of its type is fresh enough and satisfies every condition. One artifact whose conditions are false fails the clause, even when another artifact lacks a value the clause reads. This clause passes when every system in the pack requires SSO:

```json
{
  "type": "evidencepack/access-posture@v1",
  "artifacts": "all",
  "metadata_conditions": {
    "all": [
      { "path": "$.sign_in.sso_required", "op": "eq", "value": true }
    ]
  }
}
```

`cardinality` (Section 4.5) applies to the values a path returns from one artifact, and `artifacts` applies to the artifacts of the clause's type. A profile that sets `artifacts` needs epack-tool-validate v0.2.0 or later. With an earlier release, validation stops with an error on that profile.

The name `metadata_conditions` is historical. Conditions read the artifact's content.

`freshness` measures when the pack collected the artifact. To check when the evidence itself was produced, such as the date of a penetration test, use the `max_age_days` operator on a date in the content.

### 4.5 Conditions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | String | Yes | JSONPath expression rooted at `$`, read against the artifact's content |
| `op` | String | Yes | Operator (see below) |
| `value` | Any | See below | Value to compare against. Not used by `exists` and `not_exists` |
| `cardinality` | String | See below | How a path with several values is scored: `all`, `any`, or `none` |

**Paths** use JSONPath ([RFC 9535](https://www.rfc-editor.org/rfc/rfc9535)).

| Pattern | Example | Reads |
|---------|---------|-------|
| Root field | `$.provider` | A top-level field |
| Nested field | `$.policy.mfa_required` | A nested field |
| Array index | `$.accounts[0].account_id` | One element |
| Last element | `$.accounts[-1].account_id` | The last element |
| Wildcard | `$.accounts[*].storage.encryption_pct` | A field in every element |
| Recursive descent | `$..mfa_coverage_pct` | A field at any depth |

A path with a wildcard, recursive descent, filter, or slice can return several values, and its condition MUST set `cardinality`. Any other path reads at most one value, and its condition omits `cardinality`.

**Operators:**

| Operator | Passes when |
|----------|-------------|
| `eq` | The value equals `value`. Numbers compare by value, so `1` equals `1.0`. Other values compare exactly, with no conversion between strings and numbers |
| `neq` | The value does not equal `value`, by the same rules |
| `gt`, `gte`, `lt`, `lte` | The value is a number greater than, at least, less than, or at most `value`. A value that is not a number does not pass |
| `exists` | The value is present and not null |
| `not_exists` | The value is absent or null |
| `max_age_days` | The value is an ISO date (`YYYY-MM-DD`) or an RFC 3339 timestamp no more than `value` days old and not in the future. A date with no time may be up to one day in the future, to absorb time zone differences |

Every operator except `exists` and `not_exists` requires `value`. `gt`, `gte`, `lt`, `lte`, and `max_age_days` require a numeric `value`, and `max_age_days` requires it to be zero or more. Validators also accept `==`, `equals`, `!=`, `not_equals`, `>`, `>=`, `<`, and `<=` as aliases. Authors SHOULD use the names in the table.

**Absent values:** A value that is missing or null cannot pass or fail a comparison, so the condition is unknown (Section 6.3). Only `exists` and `not_exists` score absence, since absence is what they test. A profile that needs a field to be present says so with an `exists` condition.

**Cardinality:**

| Cardinality | Passes when |
|-------------|-------------|
| `all` | Every value satisfies the condition |
| `any` | At least one value satisfies the condition |
| `none` | No value satisfies the condition |

```json
{ "path": "$.accounts[*].logging.audit_log_enabled", "op": "eq", "value": true, "cardinality": "all" }
```

### 4.6 Graduated Severity

In an `any_of` requirement, the clauses without `severity` state the bar for a pass, and the clauses with `severity` grade a miss. The validator takes the first clause that passes, in order, so graded clauses go from the mildest to the most severe.

```json
{
  "id": "mfa-coverage",
  "control": "CC6.1",
  "name": "MFA is enforced for every user",
  "satisfied_by": {
    "any_of": [
      {
        "type": "evidencepack/idp-posture@v1",
        "metadata_conditions": { "all": [ { "path": "$.user_security.mfa_coverage_pct", "op": "gte", "value": 100 } ] }
      },
      {
        "type": "evidencepack/idp-posture@v1",
        "severity": "low",
        "metadata_conditions": { "all": [ { "path": "$.user_security.mfa_coverage_pct", "op": "gte", "value": 95 } ] }
      },
      {
        "type": "evidencepack/idp-posture@v1",
        "severity": "medium",
        "metadata_conditions": { "all": [ { "path": "$.user_security.mfa_coverage_pct", "op": "gte", "value": 80 } ] }
      }
    ]
  }
}
```

At 100% coverage the requirement passes. At 97% it fails with severity `low`, and at 85% with severity `medium`. Below 80% no clause passes, so it fails with severity `high`. When the identity provider artifact has no `mfa_coverage_pct`, the requirement is unknown.

In an `all_of` requirement, every clause must pass. When they all pass and some carry a severity, the requirement fails at the highest of those severities.

### 4.7 Authoring Checks

Authoring tools SHOULD check each condition path against the schema of the clause's type, so a misspelled field is caught when the profile is written. Otherwise it reads as absent in every pack and the requirement is always unknown.

## 5. Overlays

An overlay changes a profile for one pipeline or audience without editing the profile itself, so later versions of the profile still reach every pipeline that uses it.

### 5.1 Overlay Structure

```json
{
  "id": "evidencepack/hipaa-overlay@v1",
  "name": "HIPAA Overlay",
  "description": "Additional requirements for HIPAA compliance.",
  "modify": [
    {
      "id": "pentest",
      "name": "A penetration test was performed in the last 6 months",
      "category": "Security Testing",
      "satisfied_by": {
        "any_of": [
          {
            "type": "evidencepack/pentest-report@v1",
            "metadata_conditions": {
              "all": [
                { "path": "$.metadata.test_date", "op": "max_age_days", "value": 180 }
              ]
            }
          }
        ]
      }
    }
  ],
  "add": [
    {
      "id": "baa",
      "name": "A business associate agreement is offered",
      "category": "HIPAA",
      "manual": true,
      "description": "The business associate agreement offered to covered entities"
    },
    {
      "id": "hipaa-risk-assessment",
      "name": "A HIPAA risk assessment was completed in the last 12 months",
      "category": "HIPAA",
      "manual": true,
      "description": "Security risk assessment of systems that hold ePHI, completed within the last 12 months"
    }
  ]
}
```

### 5.2 Overlay Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | String | No | Identifier of the overlay |
| `name` | String | No | Human-readable name |
| `description` | String | No | What the overlay changes |
| `modify` | Array | No | Requirements to replace, matched by `id` |
| `skip` | Array of String | No | Ids of requirements to remove |
| `add` | Array | No | Requirements to append, in the form of Section 4.3 |

A `modify` entry has the requirement fields of Section 4.3 and MUST include `id` and `name`. It replaces the requirement with the same `id` entirely. Fields it leaves out become empty, so an entry restates `satisfied_by` and every other field it keeps.

### 5.3 Merge Semantics

Overlays apply in declared order, and each one sees the result of the ones before it. An overlay can therefore modify or skip a requirement an earlier overlay added. Within one overlay:

1. **`skip`** removes the requirements with the listed ids
2. **`modify`** replaces requirements by id
3. **`add`** appends requirements

The following stop validation with an error (Section 6.8):

- A `skip` or `modify` entry names an id that is not in the effective profile
- An `add` entry uses an id that is already in the effective profile

**Overlays can loosen a profile.** `skip` and `modify` can remove or relax requirements. A pack's manifest records every overlay it was validated with (Section 7.3), so a reader can see that the effective profile differs from the published one.

## 6. Semantic Validation

Semantic validation checks a pack against the effective profile. It runs after structural validation (core spec) succeeds.

### 6.1 Indexing Artifacts

The validator groups the pack's artifacts by `schema`, keeping their order in the artifact index. Artifacts without a `schema` are left out. For each artifact with a `schema`:

- Its content MUST parse as JSON. If it does not, validation stops with an error.
- When it has `collected_at`, the value MUST parse as an RFC 3339 timestamp. If it does not, validation stops with an error. The core spec requires the narrower `YYYY-MM-DDTHH:MM:SSZ` form.

### 6.2 Clause Evaluation

For each clause, the validator takes the artifacts whose `schema` equals the clause's `type`. A clause without `artifacts`, or with `artifacts: any`, is evaluated as follows.

1. If there are none, the clause is **missing**.
2. Otherwise it goes through them in order. When the clause has `freshness`, it skips an artifact with no `collected_at`, and an artifact collected more than `max_age_days` × 24 hours before the current time in UTC. For each artifact it does not skip, it evaluates the conditions against the artifact's content (Section 6.3). When they hold, the clause **passes** with that artifact and the remaining artifacts are not read.
3. If no artifact passed, the clause's outcome is the first of these that applies.
   - **unknown** (`insufficient_evidence`), when the conditions were unknown for an artifact that was not skipped
   - **fail** (`condition`), when the conditions were false for an artifact that was not skipped
   - **stale**, when an artifact was skipped as too old
   - **freshness_missing**, when artifacts were skipped only for lacking `collected_at`

An unknown outranks a definite failure on another artifact, because the artifact with absent evidence might have satisfied the clause.

**`artifacts: all`:** The validator skips artifacts for freshness by the same rule and evaluates the conditions against every artifact it does not skip. The clause's outcome is the first of these that applies.

1. **missing**, when there are no artifacts of the type
2. **fail** (`condition`), when the conditions were false for an artifact
3. **unknown** (`insufficient_evidence`), when the conditions were unknown for an artifact
4. **stale**, when an artifact was skipped as too old
5. **freshness_missing**, when an artifact was skipped for lacking `collected_at`
6. **pass**, when the conditions held for every artifact

Here a definite failure outranks an unknown. The failing artifact fails the clause whatever the absent value would have been.

### 6.3 Condition Evaluation

Evaluation is three-valued. A condition is true, false, or unknown, and so is the `all` group. The group is false when any condition is false, otherwise unknown when any condition is unknown, otherwise true.

**A path without cardinality** reads one value. A missing or null value makes the condition unknown, except under `exists` and `not_exists`. Otherwise the operator decides.

**A path with cardinality** reads every value it matches. A null value is unevaluated. When the path walks an array, an element of that array that lacks the field is unevaluated too, so a partly collected artifact cannot pass an `all` check by leaving fields out.

| Cardinality | True when | False when | Unknown when |
|-------------|-----------|------------|--------------|
| `all` | every value passes | a value fails | no value fails and one is unevaluated |
| `any` | a value passes | every value fails | no value passes and one is unevaluated |
| `none` | no value passes and none is unevaluated | a value passes | no value passes and one is unevaluated |

When the path matches no values, the condition is unknown. The exception is `none` over an array that is present and empty, which is true.

**`exists` and `not_exists` with cardinality** judge each element of the array the path walks. An element that lacks the field or holds null does not have it. `all`, `any`, and `none` then count the elements that have it. These conditions are never unknown.

### 6.4 Requirement Outcomes

**`any_of`:**

1. If a clause passes without a severity, the requirement **passes**, whatever the other clauses did.
2. Otherwise, if any clause is unknown, the requirement is **unknown**.
3. Otherwise, if a clause passed with a severity, the requirement **fails** at the severity of the first such clause.
4. Otherwise it **fails** at severity `high`.

**`all_of`:**

1. If a clause did not pass and is not unknown, the requirement **fails** at severity `high`, even when other clauses are unknown.
2. Otherwise, if any clause is unknown, the requirement is **unknown**.
3. Otherwise, if any clause passed with a severity, it **fails** at the highest of those severities.
4. Otherwise it **passes**.

A requirement that is unknown, or fails because a clause did not pass, reports the reason as its `failure_kind`. The reason is `missing`, `stale`, `freshness_missing`, `condition`, or `insufficient_evidence`. When several clauses did not pass, it reports the most informative one, preferring a condition failure. A graded failure has no `failure_kind`.

### 6.5 Overall Status

- A manual requirement is not evaluated. Its status is `manual`, and it never affects the overall status.
- The overall status is `fail` when any requirement fails, otherwise `unknown` when any requirement is unknown, otherwise `pass`.
- `summary.total` counts every requirement, manual ones included. `summary.passed`, `summary.unknown`, and `summary.manual` count requirements with those statuses. `summary.missing` counts failures whose `failure_kind` is `missing`, and `summary.failed` counts the other failures. `summary.warnings` counts the warnings of Section 6.8.
- `key_failures` lists the failed requirements whose severity is `critical` or `high`.
- `by_category` gives the same counts for each `category`.

A failing pack is a result, not an error. The validator reports it and exits successfully.

### 6.6 Asserted Support

When the pack contains control mapping artifacts (Section 1.4, [schemas/control-mapping.md](schemas/control-mapping.md)) for the profile being validated, the validator joins their entries to requirements, resolves each entry's evidence against the pack, and reports the results in the `asserted_support` output section. Mappings whose `profile.id` names a different profile are ignored.

| Entry field | Joins to |
|-------------|----------|
| `requirement` | The requirement with that `id` |
| `control` | The first requirement with that `control` |
| `type` | The one requirement whose clauses read that type. When several do, the entry joins to none |

- A mapping whose `profile.digest` differs from the digest of the profile validated is still reported. Each of its entries is marked stale, and the validator emits `MAPPING_PROFILE_MISMATCH`.
- An entry whose artifact, document, pointer, quote, or row is not in the pack, or whose join matches no requirement, is reported with `MAPPING_UNRESOLVED`. An entry that matches no requirement is reported without one.
- A malformed entry is reported with `INVALID_MAPPING_ENTRY` and skipped.
- A manual requirement with no entries produces `MANUAL_UNSUPPORTED`.

Each reported entry has a `resolution`. It is `resolved` when its target and anchor were found, `document_only` when the document was found but the anchor's extraction is absent, and `unresolved` when the target or anchor is not in the pack.

Nothing in this step changes any requirement outcome or the overall status.

### 6.7 Output Format

A validator writes one result for the run.

```json
{
  "status": "fail",
  "profile": {
    "id": "acme/security-baseline@v1",
    "name": "ACME Security Baseline",
    "version": "1.0.0",
    "digest": "sha256:9f2c41d6e8a3b7c015d94f6a2e8b1c7d3a5f0e9b8c6d4a2f1e0d9c8b7a6f5e4d"
  },
  "validated_at": "2026-11-03T12:00:00Z",
  "pack_digest": "sha256:0a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f9",
  "summary": { "total": 5, "passed": 1, "failed": 1, "missing": 0, "unknown": 0, "manual": 3, "warnings": 1 },
  "requirements": [
    {
      "id": "access-authorization",
      "name": "Sign-in requires strong authentication",
      "control": "AC-01",
      "category": "Access Control",
      "status": "pass",
      "mode": "all_of"
    },
    {
      "id": "threat-detection",
      "name": "Threats are detected on endpoints or in the cloud",
      "control": "SI-04",
      "category": "Monitoring",
      "status": "fail",
      "severity": "high",
      "failure_kind": "condition",
      "mode": "any_of",
      "artifact": "artifacts/endpoint-posture.json",
      "path": "$.protection.edr_installed_pct",
      "expected": { "op": "gte", "value": 95 },
      "actual": 88,
      "delta": -7,
      "message": "conditions not satisfied",
      "checks": [
        {
          "clause_index": 0,
          "schema": "evidencepack/endpoint-posture@v1",
          "status": "fail",
          "selected": true,
          "artifact": "artifacts/endpoint-posture.json",
          "conditions": [
            { "path": "$.protection.edr_installed_pct", "expected": { "op": "gte", "value": 95 }, "actual": 88, "delta": -7, "passed": false }
          ]
        },
        {
          "clause_index": 1,
          "schema": "evidencepack/cloud-posture@v1",
          "status": "fail",
          "artifact": "artifacts/cloud-posture.json",
          "conditions": [
            { "path": "$.accounts[*].threat_detection.enabled", "expected": { "op": "eq", "value": true }, "actual": false, "passed": false }
          ]
        }
      ]
    },
    { "id": "audit-logging", "name": "Production audit logs are monitored", "control": "AU-01", "status": "manual" },
    { "id": "security-policy", "name": "An information security policy is approved and reviewed annually", "control": "PL-01", "status": "manual" },
    { "id": "access-review", "name": "User access is reviewed every quarter", "control": "AC-02", "status": "manual" }
  ],
  "key_failures": [ { "id": "threat-detection", "name": "Threats are detected on endpoints or in the cloud", "severity": "high" } ],
  "asserted_support": {
    "acme/security-baseline@v1": [
      {
        "control": "PL-01",
        "requirement_id": "security-policy",
        "evidence": { "document_id": "doc-8f2c1a9e", "quote": { "exact": "reviewed at least annually and approved by the CISO" } },
        "label": "Information Security Policy.pdf",
        "resolution": "resolved"
      },
      {
        "control": "AU-01",
        "requirement_id": "audit-logging",
        "evidence": { "artifact": "artifacts/monitoring-export.json" },
        "label": "Alert rules export",
        "resolution": "resolved"
      }
    ]
  },
  "warnings": [ { "code": "MANUAL_UNSUPPORTED", "message": "manual requirement access-review has no asserted support" } ]
}
```

**Requirement results:**

| Field | Description |
|-------|-------------|
| `id`, `name`, `control`, `description`, `category` | Copied from the requirement |
| `status` | `pass`, `fail`, `unknown`, or `manual` |
| `severity` | For a failure, `critical`, `high`, `medium`, or `low` |
| `failure_kind` | For a failure or unknown, `missing`, `stale`, `freshness_missing`, `condition`, or `insufficient_evidence` |
| `mode` | `any_of` or `all_of` |
| `artifact`, `path`, `expected`, `actual`, `delta` | The artifact and condition that decided the outcome. `delta` is `actual` minus `value` for numbers |
| `message` | Human-readable detail |
| `checks` | One entry per clause, with its `status` (`pass`, `fail`, `unknown`, `missing`, `stale`, or `freshness_missing`), the artifact read, and each condition's result. `selected` marks the clause that decided an `any_of` outcome. The check for a clause with `artifacts: all` also has `artifacts` (see below) |

For a graded failure, the result shows the first clause's condition when it reads the same field as the clause that passed, so the reader sees the gap to a full pass.

For a clause with `artifacts: all`, the check has an `artifacts` object. This check read three access-posture artifacts, and one failed:

```json
{
  "clause_index": 1,
  "schema": "evidencepack/access-posture@v1",
  "status": "fail",
  "artifact": "artifacts/clickhouse.access-posture.json",
  "artifacts": {
    "mode": "all",
    "evaluated": 3,
    "passed": 2,
    "failed": ["artifacts/clickhouse.access-posture.json"]
  },
  "conditions": [
    { "path": "$.sign_in.mfa_required", "expected": { "op": "eq", "value": true }, "actual": false, "passed": false }
  ]
}
```

`evaluated` counts the artifacts whose conditions were evaluated, and `passed` counts the ones that satisfied them. `failed` lists the ones whose conditions were false, and it is absent when there are none. An artifact skipped for freshness is in none of these. A failing check names the first failed artifact in `artifact`. When more than one artifact passed, a passing check has no `artifact`.

### 6.8 Warnings and Errors

Warnings appear in the result's `warnings` array. They never change a requirement outcome or the overall status.

| Code | Meaning |
|------|---------|
| `MANUAL_UNSUPPORTED` | A manual requirement has no asserted support entries |
| `MAPPING_UNRESOLVED` | A control mapping entry cites an artifact, document, pointer, quote, or row not in the pack, or a requirement not in the effective profile |
| `MAPPING_PROFILE_MISMATCH` | A control mapping's profile digest differs from the profile validated |
| `INVALID_MAPPING_ENTRY` | A control mapping entry is malformed (see schemas/control-mapping.md Section 3) |

The following are errors in the run rather than outcomes. The validator reports the error and writes no result.

- The profile or an overlay cannot be read or parsed
- The effective profile breaks a rule of Section 4, such as a duplicate requirement `id`, an unknown operator, a multi-value path without `cardinality`, or an `artifacts` value other than `any` or `all`
- An overlay's `skip` or `modify` names a missing id, or its `add` reuses one (Section 5.3)
- The profile's digest differs from the one the manifest records (Section 7.3)
- A typed artifact's content is not JSON, or its `collected_at` is malformed (Section 6.1)

## 7. Loading and Locking

### 7.1 Sources

Profiles and overlays are files. A project names them by path. Naming them by registry identifier is reserved for a future version (Appendix B), and current implementations load local files only. Validators MUST NOT fetch a profile or overlay over the network without explicit consent.

### 7.2 Digests

A profile or overlay digest is `sha256:` followed by the lowercase hex SHA-256 of the file's bytes, exactly as stored. Any change to the file, including whitespace or key order, changes its digest.

### 7.3 Recording in the Pack

A builder that prepares a pack for a profile records it in the manifest `profiles` array, and its overlays in `overlays`, as `{source, digest}` entries ([pack.md](pack.md) Section 10). `source` is the path or identifier the project used.

When the manifest records one profile with a digest, the validator MUST compute the digest of the profile it loaded and stop with an error if they differ. Validators MUST NOT substitute a different profile or version for the one the manifest records. A manifest that records more than one profile is reserved for a future version, and validators stop with an error on it.

Control mappings name the profile they were made against by `id` and digest, and validators compare that digest with this one (Section 6.6).

## 8. Reference Profiles

These reference profiles demonstrate the concepts and can be used as starting points. They are published under the `evidencepack/` namespace in [profiles/](../../profiles/).

| Profile | Description |
|---------|-------------|
| `evidencepack/soc2-basic@v1` | Minimum evidence for a SOC 2 security review. Requires a SOC 2 Type II report and a penetration test from the last 12 months, and asks for the information security policy as a manual requirement. |
| `evidencepack/vendor-review@v1` | Third-party vendor assessment. Adds email DNS and TLS posture from the last 90 days to soc2-basic, and asks for an architecture diagram. Both posture types can be collected without the vendor's credentials. |
| `evidencepack/hipaa-overlay@v1` | HIPAA overlay. Tightens the `pentest` requirement to 6 months and adds the business associate agreement and a risk assessment as manual requirements. |
| `evidencepack/cloud-posture-overlay@v1` | Requires cloud, identity provider, and version control posture collected within 90 days. Also requires access and vulnerability scanning posture collected within 90 days for every system in the pack. |

The profiles reference these types, also published under `evidencepack/`:

| Type | Description |
|------|-------------|
| `evidencepack/soc2-report@v1` | SOC 2 Type I or Type II report, with the report period, auditor, and opinion as metadata |
| `evidencepack/pentest-report@v1` | Penetration test report, with the test dates, assessor, and findings by severity as metadata |
| `evidencepack/identity-center-mfa-settings@v1` | Capture of the AWS IAM Identity Center MFA settings, which no API reports |
| `evidencepack/document@v1` | Any other document, with no metadata fields |
| `evidencepack/cloud-posture@v1` | Normalized cloud account posture ([schemas/cloud-posture.md](schemas/cloud-posture.md)) |
| `evidencepack/idp-posture@v1` | Normalized identity provider posture ([schemas/idp-posture.md](schemas/idp-posture.md)) |
| `evidencepack/vcs-posture@v1` | Normalized version control posture ([schemas/vcs-posture.md](schemas/vcs-posture.md)) |
| `evidencepack/endpoint-posture@v1` | Normalized device management and endpoint security posture ([schemas/endpoint-posture.md](schemas/endpoint-posture.md)) |
| `evidencepack/access-posture@v1` | Normalized sign-in, member, admin, and credential posture of one system ([schemas/access-posture.md](schemas/access-posture.md)) |
| `evidencepack/vuln-scanning@v1` | Normalized vulnerability scanner coverage and open findings for one system ([schemas/vuln-scanning.md](schemas/vuln-scanning.md)) |
| `evidencepack/email-dns@v1` | Email authentication records for a list of domains ([schemas/email-dns.md](schemas/email-dns.md)) |
| `evidencepack/tls@v1` | TLS configuration of a list of endpoints ([schemas/tls.md](schemas/tls.md)) |

## 9. Test Vectors

> **TODO:** Test vector files are not yet available. The table below defines the conformance expectations. The `.zip` files will be published in `test-vectors/profiles/` before the spec leaves Draft status.

Conforming implementations MUST produce these results:

| Vector | Description | Expected Result |
|--------|-------------|-----------------|
| `valid-soc2-basic.zip` | Pack satisfying evidencepack/soc2-basic@v1 | Status `pass`, `security-policy` reported as `manual` |
| `missing-pentest.zip` | soc2-basic pack with no `pentest-report` artifact | `pentest` fails, `failure_kind` `missing` |
| `old-pentest.zip` | soc2-basic pack whose `test_date` is 400 days old | `pentest` fails, `failure_kind` `condition` |
| `absent-test-date.zip` | soc2-basic pack whose pentest metadata has no `test_date` | `pentest` unknown, `insufficient_evidence`, status `unknown` |
| `stale-email-dns.zip` | vendor-review pack with email DNS posture collected 120 days ago | `email-dns` fails, `failure_kind` `stale` |
| `missing-collected-at.zip` | vendor-review pack whose email DNS artifact has no `collected_at` | `email-dns` fails, `failure_kind` `freshness_missing` |
| `invalid-collected-at.zip` | Typed artifact with a malformed `collected_at` | Error, no result |
| `invalid-json-content.zip` | Typed artifact whose content is not JSON | Error, no result |
| `partial-accounts.zip` | Cloud posture with one account lacking `threat_detection.enabled`, read with `cardinality: all` | Condition unknown |
| `graded-mfa.zip` | Identity provider posture at 97% MFA, against the Section 4.6 example | Fails with severity `low` |
| `every-artifact-pass.zip` | Two access-posture artifacts with `sign_in.sso_required` true, against the Section 4.4 example | Clause passes, with no `artifact` on the check |
| `every-artifact-fail.zip` | Access-posture artifacts that pass, fail, and lack `sign_in.sso_required`, against the Section 4.4 example | Clause fails, `failure_kind` `condition` |
| `every-artifact-unknown.zip` | One access-posture artifact that passes and one that lacks `sign_in.sso_required`, against the Section 4.4 example | Clause unknown, `insufficient_evidence` |
| `digest-mismatch.zip` | Manifest profile digest differs from the loaded profile | Error, no result |
| `overlay-target-missing.zip` | Overlay modifies an id not in the profile | Error, no result |
| `hipaa-overlay.zip` | soc2-basic and hipaa-overlay, with a penetration test from 200 days ago | `pentest` fails, `failure_kind` `condition` |
| `mapping-valid.zip` | Pack with a control mapping whose entries all resolve | Asserted support reported as `resolved` |
| `mapping-unresolved.zip` | Pack with a mapping entry citing an absent artifact | `MAPPING_UNRESOLVED` warning |
| `mapping-not-verdict.zip` | Failing pack whose mapping asserts support for the failing requirement | Outcomes unchanged by the mapping |
| `manual-requirement.zip` | Profile with a manual requirement and mapped evidence | Status `manual`, asserted support reported, no `MANUAL_UNSUPPORTED` |
| `manual-unsupported.zip` | Profile with a manual requirement and no mapped evidence | `MANUAL_UNSUPPORTED` warning |

## 10. Requirements Summary

| ID | Level | Requirement |
|----|-------|-------------|
| P-001 | MUST | A validator run MUST evaluate exactly one profile, after applying its overlays in declared order |
| P-002 | MUST | An artifact MUST match a clause only when its `schema` equals the clause's `type`, byte for byte |
| P-003 | MUST | Artifacts without `schema` MUST NOT match any clause |
| P-004 | MUST | Validation MUST stop with an error when a typed artifact's content is not JSON |
| P-005 | MUST | Validation MUST stop with an error when a present `collected_at` is not an RFC 3339 timestamp |
| P-006 | MUST | A profile MUST have `id`, `name`, `version`, and at least one requirement |
| P-007 | MUST | Requirement `id` values MUST be unique within the effective profile |
| P-008 | MUST | A machine-checked requirement MUST have `id`, `name`, and `satisfied_by` with exactly one of `any_of` or `all_of`, each a non-empty array of clauses |
| P-009 | MUST | A manual requirement MUST have `manual: true`, `name`, and `id` or `control`, and MUST NOT have `satisfied_by` |
| P-010 | MUST | Every clause MUST have `type`, and a present `severity` MUST be `critical`, `high`, `medium`, or `low` |
| P-011 | MUST | Condition paths MUST be JSONPath expressions rooted at `$` |
| P-012 | MUST | A condition whose path can return several values MUST set `cardinality` to `all`, `any`, or `none` |
| P-013 | MUST | Every operator except `exists` and `not_exists` MUST have a `value`, and `gt`, `gte`, `lt`, `lte`, and `max_age_days` MUST have a numeric one |
| P-014 | MUST | A condition on a missing or null value MUST be unknown, except under `exists` and `not_exists` |
| P-015 | MUST NOT | `eq` and `neq` MUST NOT convert between strings and numbers |
| P-016 | MUST | Freshness MUST be measured from `collected_at` against the validator's current time in UTC |
| P-017 | MUST | A clause without `artifacts: all` MUST pass when one artifact of its type is fresh and satisfies all its conditions |
| P-018 | MUST | A clause without `artifacts: all` that has no passing artifact and an unknown result on an artifact it read MUST be unknown, not failed |
| P-019 | MUST | Requirement outcomes MUST follow Section 6.4 for `any_of` and `all_of` |
| P-020 | MUST | The overall status MUST be `fail` when any requirement fails, otherwise `unknown` when any is unknown, otherwise `pass` |
| P-021 | MUST NOT | Manual requirements MUST NOT be evaluated or affect the overall status |
| P-022 | MUST | Within an overlay, `skip` MUST apply first, then `modify`, then `add` |
| P-023 | MUST | A `modify` entry MUST replace the whole requirement with the same `id` |
| P-024 | MUST | Validation MUST stop with an error when an overlay's `skip` or `modify` names an id not in the effective profile, or its `add` reuses one |
| P-025 | MUST | Profile and overlay digests MUST be `sha256:` followed by the lowercase hex SHA-256 of the file bytes |
| P-026 | MUST | When the manifest records one profile with a digest, validation MUST stop with an error if the loaded profile's digest differs |
| P-027 | MUST NOT | Validators MUST NOT substitute a different profile or version for the one the manifest records |
| P-028 | MUST NOT | Validators MUST NOT fetch profiles or overlays over the network without explicit consent |
| P-029 | MUST | Type identifiers MUST follow the `namespace/name@vN` format |
| P-030 | MUST | Semantic validators MUST accept unknown artifact and manifest fields |
| P-031 | MUST NOT | Control mapping content MUST NOT affect any requirement outcome or the overall status |
| P-032 | MUST | Mapping entries MUST reference exactly one requirement (`requirement`, `control`, or `type`) and exactly one evidence target (`artifact` or `document_id`) |
| P-033 | MUST | Unresolvable mapping references MUST be reported as `MAPPING_UNRESOLVED` warnings, never errors |
| P-034 | MUST | A mapping whose profile digest differs from the validated profile MUST be reported, marked stale, with `MAPPING_PROFILE_MISMATCH` |
| P-035 | MUST | A manual requirement with no asserted support MUST produce `MANUAL_UNSUPPORTED` |
| P-036 | MUST | Validators MUST accept unknown fields in control mapping documents and entries |
| P-037 | SHOULD | Authors SHOULD keep requirement `id` values stable across profile versions |
| P-038 | SHOULD | Profiles for compliance frameworks SHOULD set `control` on each requirement |
| P-039 | SHOULD | Authoring tools SHOULD check condition paths against the schema of the clause's type |
| P-040 | MUST | A present `artifacts` on a clause MUST be `any` or `all` |
| P-041 | MUST | A clause with `artifacts: all` MUST pass only when every artifact of its type is fresh and satisfies all its conditions |
| P-042 | MUST | A clause with `artifacts: all` MUST fail when its conditions are false for an artifact it read, even when they are unknown for another |

---

## Appendix A: Reference Implementation (Informative)

> **Note:** This appendix is informative, not normative. The pseudocode illustrates one way to implement Section 6. Conformance is determined by the requirements in Section 10, not by matching this implementation.

```
function validate(pack, profileFile, overlayFiles, now):
  profile = parse(profileFile)
  if pack.manifest.profiles has one entry with a digest:
    if sha256(profileFile.bytes) != that digest:
      raise error("profile digest mismatch")            // P-026
  for overlay in overlayFiles:                          // P-001
    profile = applyOverlay(profile, parse(overlay))     // Section 5.3
  checkProfile(profile)                                 // Section 4, P-006 to P-013, P-040

  index = {}                                            // schema -> artifacts, in pack order
  for artifact in pack.artifacts:
    if not artifact.schema: continue                    // P-003
    content = parseJSON(read(artifact))                 // raises on invalid JSON, P-004
    collected = artifact.collected_at ? parseRFC3339(artifact.collected_at) : null   // P-005
    index[artifact.schema].append({ artifact, content, collected })

  result = { status: "pass", requirements: [], summary: zeroCounts() }
  for req in profile.requirements:
    if req.manual:
      result.requirements.append({ id: req.id, status: "manual" })      // P-021
      continue
    outcomes = [evaluateClause(c, index[c.type], now) for c in req.clauses]
    r = combine(req.mode, outcomes)                                     // Section 6.4
    result.requirements.append(r)
    if r.status == "fail": result.status = "fail"
    else if r.status == "unknown" and result.status == "pass": result.status = "unknown"
  result.asserted_support, result.warnings = joinMappings(pack, profile)  // Section 6.6
  return result

function evaluateClause(clause, artifacts, now):
  if artifacts is empty: return { kind: "missing" }
  if clause.artifacts == "all": return evaluateEvery(clause, artifacts, now)
  sawUnknown, sawFalse, sawStale, sawNoTimestamp = false, false, false, false
  for a in artifacts:
    if clause.max_age_days:
      if not a.collected: sawNoTimestamp = true; continue
      if now - a.collected > clause.max_age_days * 24h: sawStale = true; continue
    verdict = evaluateAll(clause.conditions, a.content, now)            // Section 6.3
    if verdict == TRUE: return { kind: "pass", severity: clause.severity, artifact: a }
    if verdict == UNKNOWN: sawUnknown = true
    else: sawFalse = true
  if sawUnknown: return { kind: "unknown" }                             // P-018
  if sawFalse: return { kind: "condition" }
  if sawStale: return { kind: "stale" }
  return { kind: "freshness_missing" }

function evaluateEvery(clause, artifacts, now):
  sawUnknown, sawFalse, sawStale, sawNoTimestamp, passed = false, false, false, false, []
  for a in artifacts:
    if clause.max_age_days:
      if not a.collected: sawNoTimestamp = true; continue
      if now - a.collected > clause.max_age_days * 24h: sawStale = true; continue
    verdict = evaluateAll(clause.conditions, a.content, now)
    if verdict == TRUE: passed.append(a)
    else if verdict == UNKNOWN: sawUnknown = true
    else: sawFalse = true
  if sawFalse: return { kind: "condition" }                             // P-042
  if sawUnknown: return { kind: "unknown" }
  if sawStale: return { kind: "stale" }
  if sawNoTimestamp: return { kind: "freshness_missing" }
  return { kind: "pass", severity: clause.severity, artifacts: passed } // P-041

function combine(mode, outcomes):
  if mode == "any_of":
    if any o.kind == "pass" and not o.severity: return pass
    if any o.kind == "unknown": return unknown
    first = first o where o.kind == "pass"
    if first: return fail(first.severity)
    return fail("high")
  else:  // all_of
    if any o.kind not in ("pass", "unknown"): return fail("high")
    if any o.kind == "unknown": return unknown
    if any o.severity: return fail(highest severity)
    return pass
```

## Appendix B: Registry Resolution (Reserved)

> **Note:** This appendix records the planned design for fetching types, profiles, and overlays from registries. No current implementation fetches from a registry, and validators load local files (Section 7.1). The design may change before it is implemented.

**Registry URL structure:**

```
GET https://registry.example.com/types/{namespace}/{name}@v{N}
GET https://registry.example.com/profiles/{namespace}/{name}@v{N}
GET https://registry.example.com/overlays/{namespace}/{name}@v{N}
```

A registry returns the definition with its digest:

```json
{
  "id": "evidencepack/soc2-report@v1",
  "digest": "sha256:789def0123456789abcdef0123456789abcdef0123456789abcdef0123456789",
  "definition": {}
}
```

Validators would compute the digest locally from the returned definition and confirm it equals the registry's.

**Network policy:** Validators MUST NOT fetch from remote registries without explicit consent, given by a configuration setting, a command-line flag, or an interactive prompt. The presence of a URL in a pack, such as a profile identifier that names a registry host, does not count as consent. With network access enabled, validators would use HTTPS only, validate TLS certificates, and cache fetched definitions by digest.

**Default registry:** The default public registry for the `evidencepack/` namespace would be `https://schemas.evidencepack.org`. Other namespaces, such as `acme/`, belong to the organization that owns them and are served from its own registry.

**Registry configuration:**

```yaml
registries:
  - url: https://schemas.evidencepack.org
    trusted: true

  - url: https://registry.acme.com
    trusted: true
    auth:
      type: bearer
      token_env: ACME_REGISTRY_TOKEN
    secrets:
      - ACME_REGISTRY_TOKEN

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
```

| Method | Description |
|--------|-------------|
| `bearer` | Static bearer token sent as `Authorization: Bearer <token>` |
| `basic` | HTTP Basic auth sent as `Authorization: Basic <base64>` |
| `oauth2` | OAuth2 client credentials flow |

**Secrets:** Registry auth follows the same model as epack tools and collectors. The `secrets` array lists the environment variables a validator may read, and auth fields name variables with an `*_env` suffix rather than holding values. Validators would reject secret names with the reserved prefixes `EPACK_*`, `LD_*`, `DYLD_*`, and `_*`, and would never log, persist, or include secrets in error messages.

**Trust:** The trust model for registries is separate from the trust model for pack attestations. A future version may sign definitions with Sigstore. Until then, trust comes from explicit registry configuration and the digests recorded in the pack (Section 7.3).
