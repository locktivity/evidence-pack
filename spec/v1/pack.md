# Evidence Pack Format

**Version:** 1.0
**Status:** Draft

## 1. Introduction

An Evidence Pack is a ZIP archive that packages evidence bytes and metadata for deterministic integrity checks.

### 1.1 Digest Format Conventions

This specification uses two digest formats for different purposes:

| Format | Example | Used For |
|--------|---------|----------|
| Prefixed | `sha256:e3b0c4...` | `pack_digest`, artifact digests — self-describing content identifiers used throughout the manifest |
| Raw hex | `e3b0c4...` | `manifest_digest`, in-toto `subject[0].digest.sha256` — matches in-toto attestation semantics where the algorithm is the map key |

The prefixed format (`algo:hex`) is self-describing and used for general content addressing. The raw hex format aligns with in-toto's digest map structure (`{"sha256": "hex"}`), where the algorithm is already expressed as the key.

## 2. File Format

### 2.1 Archive Format

An Evidence Pack MUST be a ZIP archive with the `.epack` extension. The archive MUST use the ZIP64 extension if any file exceeds 4GB or if the archive contains more than 65,535 entries.

**ZIP Metadata:** Implementations MUST treat only file paths and file bytes as authoritative. ZIP metadata fields (timestamps, extra fields, comments) MUST be ignored for integrity verification purposes.

### 2.2 Directory Structure

```
pack.epack
├── manifest.json              # REQUIRED
├── artifacts/                 # REQUIRED
│   └── {artifact-files}       # Artifact files (may use subdirectories)
└── attestations/              # OPTIONAL
    └── {attestation-files}    # Signature files
```

**Minimal example:**

```
pack.epack
├── manifest.json
└── artifacts/
  └── github/branch-protection.json
```

Artifacts MUST be stored under the `artifacts/` directory. Subdirectory structure within `artifacts/` is not prescribed; implementations MAY organize artifacts however they choose (e.g., `artifacts/github/repos.json`, `artifacts/config.json`). The manifest's artifact index is authoritative for the artifact set.

### 2.3 File Path Requirements

All file paths within the archive MUST follow these rules:

#### 2.3.1 Character Set

Paths MUST be valid UTF-8.

Paths MUST NOT contain:

- NUL bytes
- ASCII control characters (U+0000–U+001F, U+007F)
- Backslashes (`\`)

ZIP entry names referenced by `artifacts[].path` MUST be UTF-8 and MUST NOT contain NUL bytes. Verifiers MUST reject packs containing any referenced entry whose name cannot be interpreted as UTF-8.

Implementations MUST treat UTF-8 decoding errors as fatal; they MUST NOT replace invalid sequences.

`artifacts[].path` MUST match the ZIP entry name as a Unicode string exactly (codepoint-for-codepoint), with no Unicode normalization or case folding.

#### 2.3.1.1 Unicode Normalization

Paths MUST be in Unicode NFC (Canonical Decomposition, followed by Canonical Composition) form. Implementations MUST reject paths that change under NFC normalization—that is, if `NFC(path) != path`, the path is invalid.

This requirement prevents homograph attacks and ensures consistent path matching across platforms with different Unicode normalization defaults (e.g., macOS HFS+ uses NFD).

**Example:**
- Valid: `caf\u00E9` — the string "café" in NFC form (é = U+00E9, a single codepoint)
- Invalid: `cafe\u0301` — the string "café" in NFD form (e + combining acute = U+0065 U+0301) — rejected because `NFC(path) != path`

**Attestation filenames:** Files in `attestations/` MUST use the `.sigstore.json` extension and follow the naming rules in [attestation.md](attestation.md), Section 8.2.

#### 2.3.2 Structural Rules

Paths MUST:

- Use forward slashes (`/`) as path separators
- Not begin with a leading slash
- Not end with a trailing slash (for file entries; see Section 2.3.2.1)
- Not contain empty segments (`//`)
- Not contain `.` or `..` segments
- Not contain backslashes (`\`)
- Not contain Windows drive letters (e.g., `C:`)
- Not be UNC paths (e.g., `//server/share`)

#### 2.3.2.1 Directory Entries and Trailing Slashes

ZIP archives may contain explicit directory entries. Directory semantics are determined by the trailing slash:

- If a path ends with `/`, implementations MUST treat it as a directory entry
- If a path does not end with `/`, implementations MUST treat it as a file entry

Directory entries (paths ending with `/`) MUST have:
- Compressed size of 0
- Uncompressed size of 0

**Attribute consistency:** If platform-specific directory attributes are present (e.g., Unix `S_IFDIR` mode bits, DOS directory attribute), they MUST be consistent with the trailing slash:
- If explicit directory attributes indicate "directory" but the path does NOT end with `/`, implementations MUST reject the entry
- If explicit directory attributes indicate "regular file" but the path ends with `/`, implementations MUST reject the entry

When no explicit directory attributes are present, the trailing slash alone determines directory semantics. This ensures compatibility with ZIP files created by tools that do not set platform-specific attributes.

File entries (paths not ending with `/`) MUST NOT be treated as directories regardless of attributes.

Directory entries are not required; implementations MUST NOT require them (R-081). When present, they MUST satisfy these consistency rules.

**Examples:**

- Valid: `artifacts/aws/iam-policies.json`, `artifacts/Quarterly Report 2026.pdf`, `artifacts/安全/报告.json`
- Invalid: `../etc/passwd`, `artifacts//test.json`, `artifacts\\test.json`

#### 2.3.3 Length Limits

| Limit | Value |
|-------|-------|
| Maximum path length | 240 bytes (UTF-8) |
| Maximum segment length | 80 bytes (UTF-8) |

**Rationale:** The 80-byte segment limit accommodates attestation filenames, which use the pattern `{8-char-hash}.sigstore.json` (22 characters minimum). The limit provides headroom for longer identity hashes while remaining well under filesystem limits.

#### 2.3.4 Reserved Names

The following base names MUST NOT be used as path segments (case-insensitive):

`con`, `prn`, `aux`, `nul`, `com1`-`com9`, `lpt1`-`lpt9`

These are reserved device names on Windows.

**Base name extraction:** The base name is the portion of a segment before the first dot (`.`). For example, the base name of `con.txt` is `con`. If a segment has no dot, the entire segment is the base name.

Implementations MUST reject path segments where the base name matches a reserved name, regardless of any file extension. This is required because Windows maps filenames like `con.txt`, `aux.log`, and `prn.doc` to device files.

**Examples:**
- Invalid: `con`, `CON`, `con.txt`, `aux.log`, `prn.whatever`, `com1.dat`
- Valid: `icon.png` (base name is `icon`), `constant.json` (base name is `constant`)

Implementations MUST enforce reserved-name rejection on all platforms.

#### 2.3.5 Trailing Dots and Spaces

Path segments MUST NOT end with a dot (`.`) or a space (` `).

**Rationale:** Windows automatically strips trailing dots and spaces from filenames. This causes path collisions where distinct ZIP entry names resolve to the same filesystem path:
- `report.` → `report`
- `file ` → `file`
- `data. ` → `data`

This creates integrity ambiguity during extraction and can be exploited for path confusion attacks.

**Examples:**
- Invalid: `artifacts/report.`, `artifacts/file `, `artifacts/data. `
- Valid: `artifacts/report`, `artifacts/file.txt`, `artifacts/data.json`

## 3. Manifest

### 3.1 Location

The manifest MUST be located at the root of the archive as `manifest.json`.

### 3.2 Format

The manifest MUST be a valid JSON object encoded in UTF-8.

**JSON numbers:** All numbers in `manifest.json` MUST be finite JSON numbers per RFC 8259. `NaN`, `Infinity`, and `-Infinity` are not permitted. Fields defined as integers (for example, `size` or `artifacts` counts) MUST evaluate to whole numbers in the range 0..2^53-1 so they can be represented exactly across implementations. Consumers MUST accept any JSON number representation (including exponent notation and fractional mantissas like `1.5e3`) that evaluates to a whole number within this range. Producers SHOULD serialize integers without exponent notation for readability.

**Duplicate keys:** Implementations MUST reject manifests containing duplicate keys at any nesting level. Duplicate key detection MUST occur during or before JSON parsing, not after. Standard JSON parsers that silently accept duplicate keys (using last-value-wins semantics) are not sufficient; implementations MUST use a parser that detects duplicates or perform a pre-parse validation pass.

This requirement prevents attacks where duplicate keys cause different implementations to interpret the manifest differently (e.g., one parser sees the first value, another sees the last).

**Unknown fields:** Implementations MUST reject manifests containing fields not defined in this specification. This ensures forward compatibility is explicit—new fields are introduced through spec version bumps, not silently ignored. The only exception is fields explicitly marked as reserved for future use (see Section 3.4.12).

**Rationale:** Silent acceptance of unknown fields can lead to security issues where attackers add fields that some implementations process and others ignore. Strict validation ensures all implementations interpret the manifest identically.

### 3.3 Schema

```json
{
  "spec_version": "1.0",
  "stream": "example/stream",
  "generated_at": "2026-01-20T12:00:00Z",
  "pack_digest": "sha256:0000000000000000000000000000000000000000000000000000000000000000",
  "sources": [],
  "artifacts": []
}
```

**Note:** `pack_digest` values in examples are illustrative and will not verify unless recomputed from the example artifact set.

### 3.4 Fields

#### 3.4.1 spec_version

- **Type:** String
- **Required:** Yes
- **Description:** The version of this specification the pack conforms to.
- **Value:** MUST be `"1.0"` for this version of the specification.

#### 3.4.2 stream

- **Type:** String
- **Required:** Yes
- **Description:** A unique identifier for this evidence stream. The stream identifier is opaque; implementations MAY use any naming convention.

#### 3.4.3 generated_at

- **Type:** String
- **Required:** Yes
- **Description:** The timestamp when the pack was generated.
- **Format:** MUST use the exact format `YYYY-MM-DDTHH:MM:SSZ` (e.g., `2026-01-20T12:00:00Z`). No timezone offsets, fractional seconds, or other ISO 8601 variants are permitted. Implementations MUST reject timestamps not matching this exact format.

**Examples:**

- Valid: `2026-01-20T12:00:00Z`
- Invalid: `2026-01-20T12:00:00+00:00`, `2026-01-20T12:00:00.123Z`

#### 3.4.4 pack_digest

- **Type:** String
- **Required:** Yes
- **Description:** A cryptographic digest of the pack contents.
- **Format:** MUST be `sha256:` followed by 64 lowercase hexadecimal characters.
- **Computation:** See Section 5.

#### 3.4.5 sources

- **Type:** Array of Source Collector objects
- **Required:** Yes (MAY be empty)
- **Description:** List of collectors (tools, scripts, or systems) that contributed artifacts to this pack.

**Sources are informational:** The `sources` array is for display and auditing purposes only. It MUST NOT affect verification, signing, or `pack_digest` computation beyond its inclusion in the manifest. Verifiers MUST NOT reject packs based on source metadata.

Any tool, script, or workflow that produces artifacts can populate this array. The naming is intentionally generic; this is not tied to any specific plugin or extension system.

#### 3.4.6 Source Collector Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | String | Yes | Collector identifier (e.g., `github`, `aws`, `manual`) |
| `version` | String | No | Version of the collector that generated the artifacts |
| `source` | String | No | Repository path where the collector source code is hosted (e.g., `github.com/locktivity/epack-collector-aws`) |
| `commit` | String | No | Git commit SHA that built the collector binary |
| `binary_digest` | String | No | SHA256 digest of the collector binary (`sha256:` followed by 64 hex chars) |
| `artifacts` | Integer | No | Number of artifacts contributed by this collector |

**Supply chain provenance:** The `source`, `commit`, and `binary_digest` fields enable cryptographic verification of which exact collector binary produced the artifacts. The `source` field specifies the repository where the collector's source code is hosted, `commit` identifies the exact Git SHA that was built, and `binary_digest` is the cryptographic digest of the compiled binary. When combined with SLSA Level 3 provenance attestations on the collector binaries, verifiers can establish a complete chain from source code to evidence output.

#### 3.4.7 provenance

- **Type:** Object (Provenance)
- **Required:** No
- **Description:** Describes the origin and attestation chain for the pack. Present when the pack was created by merging other packs.

**Provenance alone is not trusted:** The `provenance` object helps humans and tools understand pack lineage. Provenance has no security value unless the manifest is signed by a trusted identity—when an attestation is verified, provenance is covered by the signed `manifest_digest`. Verifiers MUST NOT trust provenance from unsigned or untrusted packs.

#### 3.4.8 Provenance Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | String | Yes | Pack type: `"merged"` or `"single"` |
| `merged_at` | String | Conditional | Timestamp when merge occurred (format: `YYYY-MM-DDTHH:MM:SSZ`; required if type is `"merged"`) |
| `merged_by` | String | No | Email or identifier of who performed the merge |
| `source_packs` | Array | Conditional | Array of Source Pack objects (required if type is `"merged"`) |

#### 3.4.9 Source Pack Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `stream` | String | Yes | Stream identifier of the source pack |
| `pack_digest` | String | Yes | Pack digest of the source pack |
| `manifest_digest` | String | Yes | JCS-canonicalized SHA-256 hash of the source pack's manifest.json (64 lowercase hex chars, no prefix) |
| `artifacts` | Integer | Yes | Number of artifacts from this source |
| `embedded_attestations` | Array | No | Array of Sigstore bundles from the source pack (if source pack was signed) |

#### 3.4.10 Embedded Attestation Object

Each element of the `embedded_attestations` array is a complete Sigstore bundle from the source pack, enabling verifiers to validate signatures without fetching the original pack. Source packs may have multiple attestations (e.g., signed by different identities). See [attestation.md](attestation.md) for the full attestation specification.

Each embedded attestation is a Sigstore bundle with the following structure:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mediaType` | String | Yes | MUST be `"application/vnd.dev.sigstore.bundle.v0.3+json"` |
| `verificationMaterial` | Object | Yes | Certificate chain and transparency log proof |
| `dsseEnvelope` | Object | Yes | DSSE envelope containing the in-toto statement |

The `dsseEnvelope` contains the signed in-toto Statement with `predicateType` of `"https://evidencepack.org/attestation/v1"`. See [attestation.md](attestation.md), Section 3 for the complete bundle structure.

#### 3.4.11 artifacts

- **Type:** Array of Artifact objects
- **Required:** Yes (MAY be empty)
- **Description:** Index of all artifacts in the pack, both embedded and referenced.

#### 3.4.12 Reserved Fields for Semantic Extensibility

The following fields are reserved for semantic validation extensions. Tools MUST preserve these fields during manifest manipulation but MUST NOT require them for basic pack validation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `profile` | String | No | Identifier for a semantic profile (e.g., `evidencepack/soc2-basic@v1`). See [Semantic Extensibility](#11-semantic-extensibility). |
| `overlays` | Array | No | Array of overlay identifiers that modify the base profile. |
| `profile_lock` | Array | No | Array of `{id, digest}` objects pinning profile/overlay document digests. |

These fields enable future semantic validation without requiring changes to the core pack format. Implementations that do not support semantic validation MUST ignore these fields but MUST NOT remove them.

### 3.5 Example

```json
{
  "spec_version": "1.0",
  "stream": "published/acme/prod",
  "generated_at": "2026-01-20T15:30:00Z",
  "pack_digest": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "sources": [
    {
      "name": "github",
      "version": "1.0.0",
      "source": "github.com/locktivity/epack-collector-github",
      "commit": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "binary_digest": "sha256:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
      "artifacts": 1
    },
    {
      "name": "aws",
      "version": "1.0.0",
      "source": "github.com/locktivity/epack-collector-aws",
      "commit": "f1e2d3c4b5a6f1e2d3c4b5a6f1e2d3c4b5a6f1e2",
      "binary_digest": "sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
      "artifacts": 1
    }
  ],
  "artifacts": [
    {
      "type": "embedded",
      "path": "artifacts/github/branch-protection.json",
      "digest": "sha256:1111111111111111111111111111111111111111111111111111111111111111",
      "size": 2048,
      "content_type": "application/json",
      "schema": "github/branch-protection/v1",
      "controls": ["AC-6", "CM-3"]
    },
    {
      "type": "reference",
      "name": "soc2-type-ii",
      "uri": "https://trust.vendor.com/portal/soc2",
      "digest": "sha256:9999999999999999999999999999999999999999999999999999999999999999",
      "access": {
        "policy": "nda_required"
      },
      "metadata": {
        "document_type": "SOC2",
        "issuer": "Big4 LLP",
        "period_start": "2025-01-01",
        "period_end": "2025-12-31",
        "expires_at": "2026-12-31"
      }
    }
  ]
}
```

### 3.6 Merged Pack Example

A merged pack includes provenance information showing source packs and their attestations. Artifacts from source packs are prefixed with their stream identifier (see Section 3.7.3). The `embedded_attestations` array uses Sigstore bundle format:

```json
{
  "spec_version": "1.0",
  "stream": "acme/prod",
  "generated_at": "2026-01-22T14:00:00Z",
  "pack_digest": "sha256:0000000000000000000000000000000000000000000000000000000000000000",
  "provenance": {
    "type": "merged",
    "merged_at": "2026-01-22T14:00:00Z",
    "merged_by": "security@acme.com",
    "source_packs": [
      {
        "stream": "acme/platform/prod",
        "pack_digest": "sha256:2222222222222222222222222222222222222222222222222222222222222222",
        "manifest_digest": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
        "artifacts": 2,
        "embedded_attestations": [
          {
            "mediaType": "application/vnd.dev.sigstore.bundle.v0.3+json",
            "verificationMaterial": {
              "x509CertificateChain": { "certificates": ["..."] },
              "tlogEntries": [{ "logIndex": "123456", "inclusionProof": {} }]
            },
            "dsseEnvelope": {
              "payloadType": "application/vnd.in-toto+json",
              "payload": "...",
              "signatures": [{ "sig": "..." }]
            }
          }
        ]
      },
      {
        "stream": "acme/it/prod",
        "pack_digest": "sha256:3333333333333333333333333333333333333333333333333333333333333333",
        "manifest_digest": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
        "artifacts": 1,
        "embedded_attestations": [
          {
            "mediaType": "application/vnd.dev.sigstore.bundle.v0.3+json",
            "verificationMaterial": {
              "x509CertificateChain": { "certificates": ["..."] },
              "tlogEntries": [{ "logIndex": "123457", "inclusionProof": {} }]
            },
            "dsseEnvelope": {
              "payloadType": "application/vnd.in-toto+json",
              "payload": "...",
              "signatures": [{ "sig": "..." }]
            }
          }
        ]
      }
    ]
  },
  "sources": [],
  "artifacts": [
    {
      "type": "embedded",
      "path": "artifacts/acme/platform/prod/config.json",
      "digest": "sha256:4444444444444444444444444444444444444444444444444444444444444444",
      "size": 1024,
      "content_type": "application/json"
    },
    {
      "type": "embedded",
      "path": "artifacts/acme/platform/prod/audit.json",
      "digest": "sha256:5555555555555555555555555555555555555555555555555555555555555555",
      "size": 2048,
      "content_type": "application/json"
    },
    {
      "type": "embedded",
      "path": "artifacts/acme/it/prod/inventory.json",
      "digest": "sha256:6666666666666666666666666666666666666666666666666666666666666666",
      "size": 512,
      "content_type": "application/json"
    }
  ]
}
```

**Note:** The bundle values are abbreviated for readability. See [attestation.md](attestation.md), Section 3 for the complete Sigstore bundle structure.

### 3.7 Merged Pack Requirements

Merged packs are standard Evidence Packs that include a `provenance` object describing the source packs.

#### 3.7.1 Manifest Requirements

- If `provenance.type` is `"merged"`, the manifest MUST include `merged_at` and a non-empty `source_packs` array.
- Each `source_packs[]` entry MUST include `stream`, `pack_digest`, `manifest_digest`, and `artifacts`.
- If source pack attestations are recorded, they MUST be provided as `source_packs[].embedded_attestations` array containing complete Sigstore bundles.
- Attestations inside a merged pack's `attestations/` directory apply only to the merged pack's `manifest.json`. To verify source pack integrity, verifiers MUST verify each entry in `embedded_attestations` using the Sigstore trusted root and expected signer identities.

#### 3.7.2 Stream Uniqueness

All source pack streams MUST be unique. Implementations MUST reject merge operations where any two source packs share the same stream identifier.

When merging a pack that was itself created by a merge (`provenance.type == "merged"`), the uniqueness check MUST include all streams from its `provenance.source_packs` array. This ensures that streams are unique across the entire merged hierarchy, not just the immediate sources.

**Example of rejected merge:**

```
# Pack A: stream = "org/audit"
# Pack B: stream = "org/audit"
# Merge A + B → ERROR: duplicate stream "org/audit"
```

**Example with nested merged pack:**

```
# Pack A: stream = "org/a"
# Pack B: stream = "org/b"
# Merged M1 (from A + B): provenance.source_packs = [{stream: "org/a"}, {stream: "org/b"}]
# Pack C: stream = "org/a"
# Merge M1 + C → ERROR: duplicate stream "org/a" (from M1's provenance and C)
```

#### 3.7.3 Artifact Path Structure

When merging source packs, artifact paths are constructed as follows:

**For non-merged source packs:** Artifacts are prefixed with the source stream to create namespaced paths:

```
{source_artifact_path} → artifacts/{source_stream}/{relative_path}
```

Where `{relative_path}` is the artifact path with the leading `artifacts/` removed.

**Example:**
- Source pack stream: `org/audit`
- Source artifact: `artifacts/report.json`
- Merged path: `artifacts/org/audit/report.json`

**For already-merged source packs:** Artifact paths MUST be preserved as-is (flattened). Since already-merged packs have paths that are already stream-prefixed from their original merge, re-prefixing would create unbounded path nesting.

**Example of flattening:**

```
# Pack A (stream: org/a) has: artifacts/data.json
# Pack B (stream: org/b) has: artifacts/data.json
# Merged M1 (stream: org/merged) contains:
#   - artifacts/org/a/data.json
#   - artifacts/org/b/data.json
#
# Pack C (stream: org/c) has: artifacts/other.json
# Merge M1 + C → Final pack contains:
#   - artifacts/org/a/data.json      (preserved from M1, NOT re-prefixed)
#   - artifacts/org/b/data.json      (preserved from M1, NOT re-prefixed)
#   - artifacts/org/c/other.json     (prefixed because C is not a merged pack)
```

This flattening behavior ensures that artifact paths remain at a single level of stream prefixing regardless of how many merge operations are performed. Combined with stream uniqueness, this guarantees no path collisions.

## 4. Artifact Index

The `artifacts` array in the manifest provides a complete index of all artifacts, enabling diffing, filtering, and integrity verification without unpacking the archive.

### 4.1 Artifact Types

Artifacts are a discriminated union with two types:

| Type | Description | Digest | Implementation |
|------|-------------|--------|----------------|
| `embedded` | Bytes are stored inside the ZIP archive | Required | MUST support |
| `reference` | Bytes are stored externally (gated portals, VDRs) | Optional (informational if absent) | MAY support |

Implementations MAY choose to support only embedded artifacts. Referenced artifacts are an optional extension for documenting external evidence such as NDA-gated portals, trust centers, and vendor document repositories (VDRs).

### 4.2 Embedded Artifact

An embedded artifact has its bytes stored inside the ZIP archive.

```json
{
  "type": "embedded",
  "path": "artifacts/aws/iam-policies.json",
  "digest": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "size": 48213,
  "content_type": "application/json",
  "collected_at": "2026-01-20T14:00:00Z",
  "schema": "aws/iam-policies/v1",
  "controls": ["AC-2", "AC-6"]
}
```

#### 4.2.1 Embedded Artifact Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | String | Yes | MUST be `"embedded"` |
| `path` | String | Yes | Path within the archive (see Section 2.3) |
| `digest` | String | Yes | SHA-256 digest of file bytes: `sha256:{hex}` |
| `size` | Integer | Yes | File size in bytes |
| `content_type` | String | No | MIME type (e.g., `application/json`) |
| `display_name` | String | No | Human-readable name for the artifact |
| `description` | String | No | Human-readable description of the artifact |
| `collected_at` | String | No | Timestamp when artifact was collected (format: `YYYY-MM-DDTHH:MM:SSZ`) |
| `schema` | String | No | Schema identifier (e.g., `github/branch-protection/v1`) |
| `controls` | Array | No | Related compliance control identifiers |

All `artifacts[].path` values for embedded artifacts MUST be unique across the manifest. Uniqueness MUST be determined using Windows-canonical path comparison to ensure cross-platform portability.

#### 4.2.2 Path Uniqueness (Windows-Canonical Comparison)

To detect duplicate paths, implementations MUST normalize paths to Windows-canonical form before comparison:

1. Split the path into segments by `/`
2. For each segment:
   - Strip trailing dots (`.`)
   - Strip trailing spaces (` `)
   - Convert to lowercase
3. Rejoin segments with `/`

Two paths that produce the same canonical form MUST be rejected as duplicates.

**Rationale:** Windows filesystems are case-insensitive and strip trailing dots and spaces. Without this check, paths like `Report.json` and `report.json`, or `data.` and `data`, would collide on Windows but appear unique in the manifest. This creates integrity ambiguity and potential security issues.

**Examples of collisions (MUST reject):**
- `artifacts/Report.json` and `artifacts/report.json` → both canonicalize to `artifacts/report.json`
- `artifacts/data.` and `artifacts/data` → both canonicalize to `artifacts/data`
- `artifacts/file ` and `artifacts/file` → both canonicalize to `artifacts/file`

**Non-colliding paths (valid):**
- `artifacts/report.json` and `artifacts/report.txt` → different canonical forms

### 4.3 Referenced Artifact

> **Implementation Note:** Referenced artifacts are an OPTIONAL feature. Implementations MAY choose not to support referenced artifacts. Implementations that do not support referenced artifacts MUST reject manifests containing `type: "reference"` artifacts with a clear error message.

A referenced artifact points to bytes stored externally, typically behind access controls (NDA-gated portals, trust centers, VDRs).

Referenced artifacts are **informational pointers** that document external evidence. They are not part of the `pack_digest` computation and do not affect pack integrity verification.

#### 4.3.0 Digest Behavior

The `digest` field is **optional** for referenced artifacts:

- **If `digest` is present and bytes are fetched:** Clients MUST validate that fetched bytes match the digest. Mismatch MUST be reported as verification failure.
- **If `digest` is absent:** The reference is an informational pointer only, not cryptographically verifiable. Clients MUST clearly indicate that the reference was not verified.

Publishers who can provide stable bytes SHOULD include a digest. Many real-world scenarios (watermarked PDFs, per-recipient portals, expiring links) cannot guarantee byte stability; omitting the digest is acceptable in these cases.

**Non-normative example (status reporting):**

```json
{
  "name": "soc2-type-ii-report",
  "reference_status": "unverified",
  "note": "No digest provided"
}
```

**Example with digest (verifiable):**

```json
{
  "type": "reference",
  "name": "soc2-type-ii-report",
  "uri": "https://trust.vendor.com/portal/soc2",
  "access": { "policy": "nda_required" },
  "digest": "sha256:1111111111111111111111111111111111111111111111111111111111111111"
}
```

**Example without digest (informational only):**

```json
{
  "type": "reference",
  "name": "pentest-report-2026",
  "uri": "https://trust.vendor.com/portal/pentest",
  "access": { "policy": "nda_required" },
  "metadata": { "document_type": "pentest", "expires_at": "2027-01-15" }
}
```

#### 4.3.1 Referenced Artifact Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | String | Yes | MUST be `"reference"` |
| `name` | String | Yes | Identifier for this reference |
| `uri` | String | Yes | URI where the artifact can be accessed |
| `access` | Object | Yes | Access policy information |
| `metadata` | Object | No | Document metadata |
| `digest` | String | No | SHA-256 digest of the referenced bytes (see Section 4.3.0) |
| `controls` | Array | No | Related compliance control identifiers |

#### 4.3.2 Access Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `policy` | String | Yes | One of: `public`, `nda_required`, `customer_only`, `request_access` |

#### 4.3.3 Metadata Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `document_type` | String | No | Type of document (e.g., `SOC2`, `pentest`, `DPA`) |
| `issuer` | String | No | Organization that issued the document |
| `period_start` | String | No | Start of coverage period (format: `YYYY-MM-DD`) |
| `period_end` | String | No | End of coverage period (format: `YYYY-MM-DD`) |
| `expires_at` | String | No | When the document expires (format: `YYYY-MM-DD`) |

**Metadata timestamps are advisory:** Fields in the Metadata Object are for display and informational purposes only. They MUST NOT affect verification, canonicalization, or signing. Implementations MAY use these values for filtering or display but MUST NOT reject packs based on metadata field values.

### 4.4 External URI Handling

> **Implementation Note:** This section applies only to implementations that support referenced artifacts. Implementations that do not support referenced artifacts MAY skip this section entirely.

Clients MUST NOT automatically fetch external URIs from referenced artifacts. Referenced URIs are informational pointers only.

When a client does fetch a referenced artifact, the following requirements apply:
- Explicit user approval or a policy allowlist MUST be required
- The URI scheme MUST be `https`
- URIs with userinfo or fragments MUST be rejected
- Clients SHOULD resolve the host and reject loopback, link-local, and private network ranges (IPv4 RFC1918, IPv6 RFC4193, IPv6 link-local)
- Redirects SHOULD re-validate scheme and host restrictions before any request body or credentials are sent
- Clients MUST NOT send Authorization headers or secrets to a different origin than the original URI
- Clients MUST redact any access tokens or credentials from logs and error messages

**IP address obfuscation:** When checking resolved IP addresses against blocklists, implementations SHOULD normalize the IP address to canonical form before comparison. Implementations SHOULD handle obfuscated IP representations including:

- Decimal format (e.g., `2130706433` for `127.0.0.1`)
- Hexadecimal format (e.g., `0x7f000001` for `127.0.0.1`)
- Octal components (e.g., `0177.0.0.1` for `127.0.0.1`)
- IPv4-mapped IPv6 addresses (e.g., `::ffff:127.0.0.1`)
- IPv6 6to4 addresses with embedded IPv4 (check the embedded IPv4 portion)

**IDN hostnames:** For internationalized domain names (IDN), implementations SHOULD normalize to ASCII (Punycode) form before validation and comparison.

These restrictions prevent SSRF-style attacks and respect the "gated" nature of referenced artifacts.

**Examples:**

- Allowed: `https://example.com/doc.pdf`
- Rejected: `http://example.com/doc.pdf`, `https://user:pass@host/`, `https://example.com/doc.pdf#frag`

## 5. Integrity

### 5.1 Pack Digest Computation

The `pack_digest` provides content-addressed integrity verification for embedded artifacts.

#### 5.1.1 Algorithm

**Inputs:** embedded artifact `path` + `digest` pairs from the manifest.

**Output:** `pack_digest` formatted as `sha256:{hex}`.

1. List all embedded artifacts from the manifest
2. For each embedded artifact in the manifest:
   - Get the file bytes from the archive at `artifact.path`
   - Verify `SHA256(bytes)` matches `artifact.digest`
3. Build a canonical artifact list as UTF-8 text:
   ```
   {path}\t{digest}\n
   ```
   Where `\t` is a single tab character (0x09) and `\n` is a single newline (0x0A)
4. Sort lines using byte-wise lexicographic ordering on raw UTF-8 bytes (equivalent to `memcmp`). Implementations MUST NOT apply locale-aware collation or Unicode normalization before sorting.
5. Concatenate all sorted lines. Each entry line ends with exactly one `\n` (0x0A). No additional trailing newline or empty line is permitted after the final entry.
6. If the embedded artifact list is empty, the digest input is an empty byte string (zero bytes).
7. Compute `SHA256` of the concatenated result
8. Format as `sha256:{hex}` (lowercase hexadecimal)

#### 5.1.2 Example

Given embedded artifacts:
```
artifacts/aws/iam.json    sha256:1111111111111111111111111111111111111111111111111111111111111111
artifacts/github/repos.json    sha256:2222222222222222222222222222222222222222222222222222222222222222
```

Canonical list (sorted):
```
artifacts/aws/iam.json\tsha256:1111111111111111111111111111111111111111111111111111111111111111\n
artifacts/github/repos.json\tsha256:2222222222222222222222222222222222222222222222222222222222222222\n
```

Pack digest: `sha256:SHA256(canonical_list)`

#### 5.1.3 Excluded Content

The following are excluded from the pack digest:

- `manifest.json` (contains the digest itself)
- All files under `attestations/` (added after pack generation)
- Referenced artifacts (bytes are external)

**Rationale:** Attestations are signatures over `manifest_digest`, not the pack digest, and are stored outside the canonical list to avoid circular dependencies. Referenced artifacts are excluded because their bytes are not in the archive.

### 5.2 Verification

To verify a pack's integrity:

1. Parse `manifest.json`
2. Enumerate all ZIP entry paths and reject:
  - Any top-level entry other than `manifest.json`, `artifacts/`, or `attestations/`
  - Any file under `artifacts/` that is not listed as an embedded artifact in `manifest.json`
  - Any ZIP file entry not referenced by the manifest, except `manifest.json` and files under `attestations/`
  - Any embedded artifact listed in the manifest that does not exist in the ZIP
  - Duplicate ZIP entry names
  - Any file under `attestations/` that is not a direct child named `*.sigstore.json` (subdirectories under `attestations/` are forbidden)
  - Directory entries in the ZIP archive MAY be ignored; implementations MUST NOT require them. Only file entries are authoritative.

Only file entries are subject to the unreferenced-entry rejection rule.
3. Extract the `pack_digest`
4. For each embedded artifact:
  - Read the file bytes from the archive
  - Compute `SHA256(bytes)`
  - Verify it matches `artifact.digest`
5. Recompute the pack digest using the algorithm above
6. Compare the computed digest with the stored digest
7. If all checks pass, the pack is internally consistent

**Example failure:** If `artifacts/a.json` is listed in the manifest but missing from the ZIP, verification fails with a missing-embedded-artifact error.

Authenticity requires one of the following:

- A trusted external `pack_digest` value (for example, from `index.json`, a lockfile, or a pinned digest)
- A valid attestation chain from a trusted key verifying the `manifest_digest`

### 5.3 Security Properties

The `pack_digest` provides the following security guarantees:

#### 5.3.1 What pack_digest Guarantees

| Property | Description |
|----------|-------------|
| **Embedded artifact integrity** | Any tampering with embedded artifact bytes is detected because each artifact's digest is verified individually |
| **Artifact set integrity** | Changes to the set of embedded artifacts (additions, removals, renames) are detected because the canonical list changes |
| **Deterministic verification** | The same pack contents always produce the same digest, enabling reproducible verification |

#### 5.3.2 What pack_digest Does NOT Cover

| Exclusion | Rationale |
|-----------|-----------|
| **Referenced artifacts** | By design, referenced artifact bytes are stored externally. The `pack_digest` does not verify their contents. If a referenced artifact includes a `digest` field and the bytes are fetched, verifiers MUST validate the digest. |
| **ZIP metadata** | Timestamps, comments, and extra fields in the ZIP archive are not covered. Only file paths and file bytes are authoritative. |
| **Attestations** | Files under `attestations/` are excluded to avoid circular dependencies. Attestations sign the `manifest_digest` instead. |

#### 5.3.3 Verification Recommendations

Implementations SHOULD:

1. Validate all artifact paths against the path rules (Section 2.3) before reading from the archive
2. Reject packs with `pack_digest` mismatches as potentially tampered
3. When referenced artifacts include digests and are fetched, clearly indicate whether verification succeeded, failed, or was not verifiable

## 6. Attestations

### 6.1 Location

Attestation files MUST be placed in the `attestations/` directory at the root of the archive.

### 6.2 Attestation Naming

Attestation files use Sigstore bundles and MUST be named using the pattern:

```
{identity-hash}.sigstore.json
```

Where `identity-hash` is the first 8 characters of the SHA-256 hash of the certificate's subject identity (email or URI).

**Examples:**

```
attestations/
├── a1b2c3d4.sigstore.json    # security@acme.com
├── e5f6g7h8.sigstore.json    # auditor@thirdparty.com
└── i9j0k1l2.sigstore.json    # github.com/org/repo workflow
```

See [attestation.md](attestation.md), Section 8 for the complete naming rules and Sigstore bundle format.

### 6.3 Multiple Attestations

A pack MAY contain multiple attestations from different signers. Each attestation MUST be in a separate file named per Section 6.2. This allows multiple parties to sign the same pack without conflicts.

### 6.4 What Attestations Sign

Attestations sign the `manifest_digest` — a hash of the JCS-canonicalized `manifest.json` content. This allows attestations to be added to a pack after creation without invalidating existing signatures.

The `manifest_digest` commits to the `pack_digest` (which is stored in the manifest), so verifying an attestation transitively verifies embedded artifact integrity. However, verifiers MUST still compute and compare the `pack_digest` to detect tampering.

See [attestation.md](attestation.md), Section 4.1.2 for manifest digest computation.

#### 6.4.1 Design Rationale: Two Digests

The specification uses two separate digests for different purposes:

| Digest | Computed From | Purpose |
|--------|---------------|---------|
| `pack_digest` | Canonical list of embedded artifacts | Content-addressed identifier for the artifact set; enables deduplication and quick integrity checks |
| `manifest_digest` | JCS-canonicalized `manifest.json` | Stable signing target; allows attestations to be added without changing the signed content |

**Why not sign `pack_digest` directly?**

Signing `pack_digest` directly would mean adding an attestation changes the pack contents, which would invalidate the `pack_digest`, which would invalidate the signature. The two-digest model breaks this circular dependency:

1. Pack is created → `pack_digest` computed and stored in manifest
2. Manifest is finalized → `manifest_digest` computed
3. Attestation signs `manifest_digest` → attestation added to pack
4. `pack_digest` unchanged (attestations excluded from computation)

**What each digest defends against:**

- `pack_digest`: Detects tampering with embedded artifact bytes or the artifact set (additions, removals, renames)
- `manifest_digest`: Binds the signer's identity to the complete manifest content, including `pack_digest`, metadata, and referenced artifacts

### 6.5 Attestation Verification Requirements

Before treating an attestation as valid, verifiers MUST:

1. Recompute `manifest_digest` from `manifest.json` using JCS (RFC 8785) and confirm it matches the attestation's subject digest
2. Verify the pack's `pack_digest` per Section 5.2
3. Verify the Sigstore bundle per [attestation.md](attestation.md), Section 7 (includes certificate chain validation, transparency log proof, and signature verification)

## 7. Security Considerations

### 7.1 Path Traversal

Implementations MUST validate that extracted paths do not escape the intended directory:

- Reject paths containing `..` segments
- Reject paths containing backslashes
- Reject absolute paths (starting with `/`)
- Validate paths match the allowed character set

**Path comparison:** All path comparisons MUST be codepoint-for-codepoint on Unicode strings after UTF-8 decoding. No percent-decoding, Unicode normalization, or case folding is permitted.

**Case sensitivity:** Paths may contain any valid UTF-8 characters including uppercase letters (Section 2.3.1). Implementations MUST NOT apply case folding during path comparison. On case-insensitive filesystems, implementations SHOULD warn if a pack contains paths that would collide (e.g., `artifacts/Test.json` and `artifacts/test.json`). Implementations MUST reject packs containing Windows reserved device names (Section 2.3.4) on all platforms to ensure cross-platform portability.

### 7.2 Size Limits

To prevent denial-of-service via excessively large packs, implementations MUST enforce size limits.

#### 7.2.1 Required Limits

| Limit | Default Value | Rationale |
|-------|---------------|-----------|
| Maximum artifact size | 100 MB | Prevents single large files from exhausting memory |
| Maximum pack size | 2 GB | Reasonable upper bound for evidence collections |
| Maximum artifact count | 10,000 | Prevents file handle exhaustion |

Implementations MUST enforce limits for maximum artifact size, pack size, and artifact count. The defaults above are recommended but may be configured.

#### 7.2.2 Hardening Limits (Recommended)

Implementations SHOULD enforce additional limits for hardening:

| Limit | Default Value | Rationale |
|-------|---------------|-----------|
| Maximum download size | 1 GB | Prevents network exhaustion during registry pulls |
| Maximum compression ratio | 100:1 | Mitigates zip bomb attacks |

Implementations MUST track total uncompressed bytes during extraction and MUST abort immediately when any limit is exceeded. Implementations MUST NOT buffer the entire uncompressed content in memory before checking limits; streaming extraction with byte counting is required.

#### 7.2.3 Extraction Accounting

When tracking bytes during extraction, implementations MUST account for **delta bytes** (bytes written in each callback or chunk), NOT cumulative bytes. If the underlying ZIP library reports cumulative bytes extracted, implementations MUST convert to delta by subtracting the previous cumulative value.

Incorrect accounting (e.g., double-counting due to cumulative vs. delta confusion) can allow zip bombs to bypass size limits.

#### 7.2.4 Configuration

Implementations SHOULD allow configuration of these limits via configuration file:

```yaml
limits:
  pack:
    max_size: 2147483648      # 2 GB in bytes
    max_artifacts: 10000
  artifact:
    max_size: 104857600       # 100 MB in bytes
  download:
    max_size: 1073741824      # 1 GB in bytes
    timeout: 5m
```

Implementations MUST enforce minimums at or above the following values and MUST NOT allow limits to be disabled entirely:

| Limit | Minimum Value |
|-------|---------------|
| Maximum artifact size | 1 MB |
| Maximum pack size | 10 MB |
| Maximum artifact count | 100 |

### 7.3 Archive Entry Types

Implementations MUST reject archive entries that are symlinks or device files. Implementations MUST check the entry type before extraction begins, not after files are written.

**Symlinks:** Symlinks are reliably detectable via ZIP mode bits. Implementations MUST reject any entry with symlink mode.

**Hard links:** ZIP format does not have a standardized hard link representation. Detection is unreliable across ZIP libraries and tools. Implementations MUST NOT attempt to detect hard links via ZIP metadata. Instead, implementations MUST:

1. Extract all file entries as regular files (never create actual hard links on the filesystem)
2. Rely on path validation to prevent path-based attacks
3. Rely on total uncompressed size limits to prevent disk exhaustion from multiple entries pointing to the same data

**Device files and special entries:** Implementations MUST reject entries with device file mode bits or other non-regular file types.

Implementations MUST NOT follow symbolic links when writing extracted content. When opening files during extraction, implementations SHOULD use `O_NOFOLLOW` or equivalent platform mechanisms to prevent symlink following where available.

Implementations SHOULD reject AppleDouble and resource fork entries (e.g., `__MACOSX/` and files prefixed with `._`).

### 7.4 Duplicate Paths

Implementations MUST reject archives that contain multiple entries for the same path (compared byte-for-byte).

### 7.5 Extraction Safety

Implementations SHOULD extract to a temporary directory and atomically rename the completed directory into place after all validation passes.

### 7.6 Permissions and Executables

Implementations SHOULD ignore ZIP permission bits and apply safe defaults (e.g., `0644` for files, `0755` for directories).

### 7.7 Logging and Secrets

Implementations MUST NOT log access tokens, Authorization headers, or credentials embedded in URLs. Implementations MUST redact sensitive query parameters and headers before writing logs or error messages.

### 7.8 External References

> **Implementation Note:** This section applies only to implementations that support referenced artifacts (see Section 4.3).

Referenced artifacts point to external URIs. Implementations that support referenced artifacts:

- MUST NOT automatically fetch external URIs
- MUST require explicit user approval or a policy allowlist before fetching
- MUST validate URIs using the restrictions in Section 4.4

### 7.9 Provenance and Embedded Attestations

For merged packs, embedded attestations (if present) are Sigstore bundles that can be verified independently using the Sigstore trusted root.

The Sigstore bundle contains all verification material (certificate chain, transparency log proof), and verification uses the public Sigstore trusted root (Fulcio CA, Rekor public key). No external key fetching is required.

Verifiers SHOULD configure expected signer identities (email addresses or OIDC URIs) to ensure embedded attestations come from trusted sources.

See [attestation.md](attestation.md), Section 7 for the complete verification procedure.

## 8. IANA Considerations

### 8.1 Media Type (Appendix)

The following media type is proposed for Evidence Packs:

- **Type name:** application
- **Subtype name:** vnd.evidence-pack+zip
- **File extension:** .epack

**Note:** This media type is proposed but not yet formally registered with IANA.

## 9. Test Vectors

Conforming implementations MUST pass the test vectors in the `/test-vectors/` directory of the specification repository. Test vectors are provided for:

| Test | Input | Expected Output |
|------|-------|-----------------|
| **Pack digest (empty)** | Zero embedded artifacts | `sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| **Pack digest (sorting)** | Artifacts in non-sorted order | Digest matches sorted canonical form |
| **Manifest digest** | Sample `manifest.json` | JCS-canonicalized bytes → SHA-256 |
| **Attestation verification** | Sample Sigstore bundle | Valid bundle verification against trusted root |
| **Timestamp validation** | Various timestamp formats | Strict format acceptance/rejection |

Test vectors include:
- Sample manifest.json files with expected `manifest_digest` values
- Sample Sigstore bundles with expected verification results
- Edge cases for path sorting (numeric prefixes, dots, dashes, underscores, segment length boundaries)

Implementations that cannot verify the provided test vectors are non-conforming.

## 10. Semantic Extensibility

This section defines reserved fields and rules for semantic validation extensions. The core pack format remains stable; semantic features are opt-in.

### 10.1 Reserved Fields

The manifest MAY include the following fields for semantic validation:

| Field | Type | Description |
|-------|------|-------------|
| `profile` | String | Identifier for a semantic profile |
| `overlays` | Array of Strings | Overlay identifiers that modify the base profile |
| `profile_lock` | Array of Objects | Pinned digests for profile/overlay documents |

### 10.2 Identifier Shape

Profile and overlay identifiers MUST follow this format:

```
Format: namespace/name@vN
```

Where:
- `namespace` and `name` are ASCII alphanumeric with hyphens and underscores (no leading/trailing hyphens)
- `vN` is a positive integer version (no leading zeros)

**Examples:** `evidencepack/soc2-basic@v1`, `acme/internal-policy@v2`

Implementations MUST treat identifiers as case-stable; comparisons are byte-for-byte on UTF-8.

### 10.3 Profile Lock

The `profile_lock` array pins specific versions of profile and overlay documents:

```json
{
  "profile_lock": [
    { "id": "evidencepack/soc2-basic@v1", "digest": "sha256:abc123..." },
    { "id": "evidencepack/hipaa-overlay@v1", "digest": "sha256:def456..." }
  ]
}
```

Each entry contains:
- `id`: The profile or overlay identifier
- `digest`: SHA-256 digest of the canonical document bytes (`sha256:` + 64 hex chars)

### 10.4 Immutability Hook

Tools that manipulate manifests (builders, mergers, converters) MUST NOT silently substitute profile or overlay identifiers. If a declared identifier cannot be resolved, tools MUST fail with an explicit error rather than falling back to a different version or removing the field.

This ensures that semantic contracts declared by the pack creator are preserved throughout the pack lifecycle.

### 10.5 Companion Specification

The detailed semantics of profiles, overlays, and validation are defined in the **Profiles & Semantic Validation** companion specification. This companion spec is versioned independently and may be updated without changes to the core pack format.

The companion spec defines:
- Profile document schema and validation rules
- Type system for artifact schemas
- Overlay composition semantics
- Validation finding codes and severity levels

## 11. References

- [RFC 1951](https://www.rfc-editor.org/rfc/rfc1951) — DEFLATE Compressed Data Format
- [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) — The JSON Data Interchange Format
- [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785) — JSON Canonicalization Scheme (JCS)
- [attestation.md](attestation.md) — Sigstore attestation format and verification
