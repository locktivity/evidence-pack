# Control Mapping Schema

**Schema Identifier:** `evidencepack/control-mapping@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

A control mapping records the publisher's assertions that specific evidence in a pack supports specific requirements of a profile. It covers requirements that no machine check evaluates, such as a written policy, a monitoring export, or a signed agreement, and it tells a consumer what the publisher offers for each control.

Control mappings are **assertions, not verification results**. They are authored by the pack publisher and sealed and signed like any other artifact. Following the same stance the core spec takes for `sources` (pack.md Section 3.4.5), mapping content MUST NOT affect verification, canonicalization, signing, or semantic validation outcomes.

A control mapping is an ordinary embedded artifact. It requires no changes to the core pack format: it lives under `artifacts/`, appears in the manifest artifact index with this schema identifier in its `schema` field, is covered by `pack_digest`, and is bound by any attestation over the manifest.

```json
{
  "type": "embedded",
  "path": "artifacts/control-mappings.json",
  "digest": "sha256:...",
  "size": 1482,
  "schema": "evidencepack/control-mapping@v1",
  "content_type": "application/json"
}
```

A pack MAY contain multiple control mapping artifacts (for example, one per declared profile). Consumers aggregate their entries.

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema_version", "captured_at", "profile", "mappings"],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "1.0.0"
    },
    "captured_at": {
      "type": "string",
      "format": "date-time",
      "description": "When this mapping document was produced (YYYY-MM-DDTHH:MM:SSZ)"
    },
    "profile": {
      "type": "object",
      "required": ["id"],
      "properties": {
        "id": {
          "type": "string",
          "pattern": "^[a-z0-9][a-z0-9_-]{0,62}/[a-z0-9][a-z0-9_-]{0,62}@v[1-9][0-9]*$",
          "description": "The profile these mappings assert against"
        },
        "digest": {
          "type": "string",
          "pattern": "^sha256:[0-9a-f]{64}$",
          "description": "JCS digest of the exact profile version mapped against (same computation as profile_lock)"
        }
      }
    },
    "mappings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["evidence"],
        "properties": {
          "requirement": {
            "type": "string",
            "description": "The requirement's `id` in the referenced profile"
          },
          "control": {
            "type": "string",
            "description": "The requirement's control identifier in the referenced profile"
          },
          "type": {
            "type": "string",
            "pattern": "^[a-z0-9][a-z0-9_-]{0,62}/[a-z0-9][a-z0-9_-]{0,62}@v[1-9][0-9]*$",
            "description": "The requirement's type identifier, for simple requirements without a control"
          },
          "evidence": {
            "type": "object",
            "properties": {
              "artifact": {
                "type": "string",
                "description": "Path of an embedded artifact in this pack (e.g. artifacts/idp-posture.json)"
              },
              "pointer": {
                "type": "string",
                "description": "RFC 6901 JSON Pointer into the artifact's JSON content"
              },
              "document_id": {
                "type": "string",
                "description": "Stable document identifier, resolved through a document index artifact in this pack"
              },
              "quote": {
                "type": "object",
                "required": ["exact"],
                "properties": {
                  "exact": { "type": "string", "description": "The cited passage, verbatim" },
                  "prefix": { "type": "string", "description": "Text immediately before the passage, for disambiguation" },
                  "suffix": { "type": "string", "description": "Text immediately after the passage, for disambiguation" }
                }
              },
              "text_position": {
                "type": "object",
                "required": ["start", "end"],
                "properties": {
                  "start": { "type": "integer", "minimum": 0 },
                  "end": { "type": "integer", "minimum": 0 }
                },
                "description": "Byte offsets of the quoted passage in the document's extracted text artifact"
              },
              "table": {
                "type": "object",
                "required": ["where"],
                "properties": {
                  "sheet": { "type": "string", "description": "Sheet name in the document's tabular extraction" },
                  "where": {
                    "type": "object",
                    "minProperties": 1,
                    "description": "Column name to value pairs identifying the cited row(s)"
                  },
                  "columns": {
                    "type": "array",
                    "items": { "type": "string" },
                    "description": "The columns being cited; all columns if absent"
                  },
                  "range": { "type": "string", "description": "Positional cell range in this sealed version (e.g. B14:E14)" }
                }
              }
            }
          },
          "label": { "type": "string", "description": "Display name for the cited evidence" },
          "note": { "type": "string", "description": "The publisher's rationale, free text" },
          "mapped_at": { "type": "string", "format": "date-time" },
          "mapped_by": { "type": "string", "description": "Who recorded the mapping (e.g. an email address)" }
        }
      }
    }
  }
}
```

## 3. Entry Constraints

- Each entry MUST reference exactly one requirement, via exactly one of `requirement`, `control`, or `type`. `requirement` joins to the requirement with the matching `id` in the referenced profile and is the preferred form; `control` joins to the requirement with the matching `control` value; `type` joins to the simple requirement with that `type`.
- Each entry's `evidence` MUST reference exactly one target: `artifact` or `document_id`, not both, not neither.
- Anchors are optional. An entry with no `pointer`, `quote`, or `table` cites the whole artifact or document, which is the common form for policies and other documents reviewed in full.
- `pointer` is only meaningful with `artifact` and only when the artifact's content is JSON.
- `quote`, `text_position`, and `table` are only meaningful with `document_id`. An entry MUST NOT carry both `quote` and `table`. `text_position` MUST NOT appear without `quote`.
- Entries violating these constraints are malformed. Consumers report them (see Section 5) and skip them; they never fail the pack.
- Consumers MUST accept unknown fields in mapping documents and entries.

## 4. Resolution

Mapping entries point at pack contents. Resolution is best-effort and display-oriented; it never affects verification.

**Artifact references.** `artifact` resolves against the manifest's embedded artifact paths. When `pointer` is present, it resolves against the artifact's parsed JSON content per RFC 6901 and names exactly one location. Query expressions are not supported.

**Document references.** `document_id` resolves through an artifact in the same pack whose `schema` identifies a document index (such as `evidencepack/document-index@v1`). The index supplies the document's files: its shipped form, and, when present, extraction artifacts for text and tabular content.

**Quote anchors.** `quote.exact` resolves by locating the passage in the document's extracted text artifact, using `prefix` and `suffix` to disambiguate repeated passages and `text_position` as the exact secondary anchor for this sealed version. When no text extraction is present, the entry resolves at document level and the quote is display-only.

**Table anchors.** `where` resolves by matching column values against rows of the document's tabular extraction, scoped to `sheet` when given; `range` is the positional anchor for this sealed version. When no tabular extraction is present, the entry resolves at document level and the table anchor is display-only.

**Resolution states.** An entry is *resolved* when its target (and anchor, where an applicable extraction exists) is found; *document-level* when the document is found but the anchor's extraction is absent; *unresolved* when the artifact or document is not in the pack, a pointer names a location the artifact does not carry, or a quote or row is absent from a present extraction. Consumers report unresolved entries as unresolved and MUST NOT silently omit them; unresolved entries are warnings, not errors (Section 5).

Entries resolve against digested content. Changing either the mapping document or a cited artifact changes their digests and fails pack verification, so a resolved citation records what the publisher pointed at and what was there.

## 5. Semantics for Validators

Semantic validators that support control mappings (see profiles.md Sections 1.4 and 6.3) join entries to the requirements of the referenced profile and report them as *asserted support* beside their computed findings. Any requirement may receive entries, whether its satisfaction is machine-checked (`type`, `satisfied_by`) or manual. The rules:

- Mapping content MUST NOT change any finding, any requirement outcome, or the pack's pass/fail result.
- A mapping whose `profile.digest` differs from the digest of the profile actually validated MUST still be reported, marked stale, with a `MAPPING_PROFILE_MISMATCH` warning.
- Unresolved entries produce a `MAPPING_UNRESOLVED` warning; malformed entries produce an `INVALID_MAPPING_ENTRY` warning.
- Control mapping artifacts are exempt from `UNTYPED_ARTIFACT` and `UNKNOWN_ARTIFACT` warnings.

Validators that do not support control mappings see an ordinary embedded artifact and are unaffected.

## 6. Example

```json
{
  "schema_version": "1.0.0",
  "captured_at": "2026-08-28T14:02:11Z",
  "profile": {
    "id": "acme/security-baseline@v1",
    "digest": "sha256:9f2c41d6e8a3b7c015d94f6a2e8b1c7d3a5f0e9b8c6d4a2f1e0d9c8b7a6f5e4d"
  },
  "mappings": [
    {
      "requirement": "access-authorization",
      "evidence": {
        "artifact": "artifacts/idp-posture.json",
        "pointer": "/policy/phishing_resistant_required_for_privileged"
      },
      "label": "Phishing-resistant MFA required for privileged users",
      "note": "Sign-on policy requires phishing-resistant factors for every admin role.",
      "mapped_at": "2026-08-12T09:30:00Z",
      "mapped_by": "dana@acme.com"
    },
    {
      "control": "PL-01",
      "evidence": {
        "document_id": "doc-8f2c1a9e",
        "quote": {
          "exact": "reviewed at least annually and approved by the CISO",
          "prefix": "This policy is ",
          "suffix": ". Exceptions require"
        },
        "text_position": { "start": 4812, "end": 4863 }
      },
      "label": "Information Security Policy.pdf",
      "mapped_at": "2026-08-12T09:34:00Z",
      "mapped_by": "dana@acme.com"
    },
    {
      "control": "AC-02",
      "evidence": {
        "document_id": "doc-4b7e2c11",
        "table": {
          "sheet": "Q2 Review",
          "where": { "Employee": "jsmith@acme.com" },
          "columns": ["Access reviewed", "Reviewer", "Review date"],
          "range": "B14:E14"
        }
      },
      "label": "Q2 Access Review.xlsx",
      "note": "Sample sign-off row from the quarterly access review.",
      "mapped_at": "2026-08-19T16:05:00Z",
      "mapped_by": "dana@acme.com"
    }
  ]
}
```
