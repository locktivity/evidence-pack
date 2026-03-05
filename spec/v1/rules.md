# Evidence Pack Specification Rules

**Version:** 1.0

This document consolidates all normative requirements from the Evidence Pack specification suite. Each rule has a unique identifier for traceability.

## Requirement Levels

| Level | Meaning |
|-------|---------|
| **MUST** | Absolute requirement; non-conforming implementations are invalid |
| **MUST NOT** | Absolute prohibition |
| **SHOULD** | Recommended; deviation requires justification |
| **SHOULD NOT** | Discouraged; use requires justification |
| **MAY** | Optional feature |

---

## Pack Format (pack.md)

### Archive Format

| ID | Level | Requirement |
|----|-------|-------------|
| R-001 | MUST | Evidence Pack MUST be a ZIP archive with `.epack` extension |
| R-002 | MUST | Archive MUST use ZIP64 extension if any file exceeds 4GB or archive contains more than 65,535 entries |
| R-003 | MUST | Implementations MUST treat only file paths and file bytes as authoritative |
| R-004 | MUST | ZIP metadata fields (timestamps, extra fields, comments) MUST be ignored for integrity verification |

### Directory Structure

| ID | Level | Requirement |
|----|-------|-------------|
| R-005 | MUST | `manifest.json` MUST be present at archive root |
| R-006 | MUST | `artifacts/` directory MUST be present |
| R-007 | MUST | Artifacts MUST be stored under the `artifacts/` directory |
| R-008 | MAY | `attestations/` directory MAY be present |

### File Path Requirements

| ID | Level | Requirement |
|----|-------|-------------|
| R-009 | MUST | Paths MUST be valid UTF-8 |
| R-010 | MUST NOT | Paths MUST NOT contain NUL bytes |
| R-011 | MUST NOT | Paths MUST NOT contain ASCII control characters (U+0000–U+001F, U+007F) |
| R-012 | MUST NOT | Paths MUST NOT contain backslashes (`\`) |
| R-013 | MUST | Paths MUST use forward slashes (`/`) as path separators |
| R-014 | MUST NOT | Paths MUST NOT begin with a leading slash |
| R-015 | MUST NOT | Paths MUST NOT end with a trailing slash (for file entries) |
| R-345 | MUST NOT | Paths MUST NOT contain Windows drive letters (e.g., `C:`) |
| R-346 | MUST NOT | Paths MUST NOT be UNC paths (e.g., `//server/share`) |
| R-321 | MUST | If path ends with `/`, implementations MUST treat it as directory entry |
| R-322 | MUST | Directory entries (paths ending with `/`) MUST have compressed and uncompressed size of 0 |
| R-323 | MUST | If explicit directory attributes indicate directory but path does NOT end with `/`, MUST reject |
| R-347 | MUST | If explicit directory attributes indicate regular file but path ends with `/`, MUST reject |
| R-016 | MUST NOT | Paths MUST NOT contain empty segments (`//`) |
| R-017 | MUST NOT | Paths MUST NOT contain `.` or `..` segments |
| R-018 | MUST | Maximum path length is 240 bytes (UTF-8) |
| R-019 | MUST | Maximum segment length is 80 bytes (UTF-8) |
| R-020 | MUST NOT | Reserved Windows names (con, prn, aux, nul, com1-9, lpt1-9) MUST NOT be used as complete path segments |
| R-021 | MUST | Implementations MUST enforce reserved-name rejection on all platforms |
| R-022 | MUST | Files in `attestations/` MUST follow the naming rules in Section 6.2 |
| R-317 | MUST | Paths MUST be in Unicode NFC form |
| R-318 | MUST | Implementations MUST reject paths that change under NFC normalization |

### Manifest

| ID | Level | Requirement |
|----|-------|-------------|
| R-023 | MUST | Manifest MUST be located at archive root as `manifest.json` |
| R-024 | MUST | Manifest MUST be a valid JSON object encoded in UTF-8 |
| R-025 | MUST | All numbers in `manifest.json` MUST be finite JSON numbers per RFC 8259 |
| R-026 | MUST NOT | `NaN`, `Infinity`, and `-Infinity` are not permitted in manifest |
| R-027 | MUST | Integer fields MUST evaluate to whole numbers (the parsed value must have no fractional component) |
| R-028 | MUST | Integer fields MUST be in the range 0..2^53-1 |
| R-029 | MUST | Consumers MUST accept any JSON number representation (including exponent notation and fractional mantissas like `1.5e3`) that evaluates to a whole number in range |
| R-030 | SHOULD | Producers SHOULD serialize integers without exponent notation |
| R-319 | MUST | Implementations MUST reject manifests containing duplicate keys at any nesting level |
| R-320 | MUST | Duplicate key detection MUST occur during or before JSON parsing, not after |

### Manifest Fields

| ID | Level | Requirement |
|----|-------|-------------|
| R-031 | MUST | `spec_version` is required and MUST be `"1.0"` for this version |
| R-032 | MUST | `stream` is required |
| R-033 | MUST | `generated_at` is required |
| R-034 | MUST | `generated_at` MUST use exact format `YYYY-MM-DDTHH:MM:SSZ` |
| R-035 | MUST | Implementations MUST reject timestamps not matching the exact format |
| R-036 | MUST | `pack_digest` is required |
| R-037 | MUST | `pack_digest` MUST be `sha256:` followed by 64 lowercase hexadecimal characters |
| R-038 | MUST | `sources` array is required (MAY be empty) |
| R-039 | MUST NOT | `sources` array MUST NOT affect verification, signing, or `pack_digest` computation |
| R-040 | MUST NOT | Verifiers MUST NOT reject packs based on source metadata |
| R-041 | MUST | `artifacts` array is required (MAY be empty) |

### Semantic Extensibility (Reserved Fields)

| ID | Level | Requirement |
|----|-------|-------------|
| R-028a | MUST | Tools MUST preserve `profile`, `overlays`, and `profile_lock` fields during manifest manipulation |
| R-028b | MUST NOT | Tools MUST NOT require reserved semantic fields for basic pack validation |
| R-028c | MUST | Profile and overlay identifiers MUST follow the format `namespace/name@vN` |
| R-028d | MUST | Identifier comparisons MUST be byte-for-byte on UTF-8 (case-stable) |
| R-028e | MUST NOT | Tools MUST NOT silently substitute declared profile or overlay identifiers |
| R-028f | MUST | If a declared identifier cannot be resolved, tools MUST fail with an explicit error |

### Provenance

| ID | Level | Requirement |
|----|-------|-------------|
| R-042 | MUST NOT | Verifiers MUST NOT trust provenance from unsigned or untrusted packs |
| R-043 | MUST | If `provenance.type` is `"merged"`, manifest MUST include `merged_at` and non-empty `source_packs` array |
| R-044 | MUST | Each `source_packs[]` entry MUST include `stream`, `pack_digest`, `manifest_digest`, and `artifacts` |
| R-044a | MUST | `source_packs[].manifest_digest` MUST be 64 lowercase hexadecimal characters (no algorithm prefix); see Section 1.1 of pack.md |
| R-045 | MUST | If source pack attestations are recorded, they MUST be provided as `embedded_attestations` array containing complete Sigstore bundles |
| R-046 | MUST NOT | Verifiers MUST NOT treat the merger's attestation (in `attestations/`) as proof of source pack integrity; to verify source pack integrity, verifiers MUST verify each entry in `embedded_attestations` using the Sigstore trusted root |

### Embedded Artifacts

| ID | Level | Requirement |
|----|-------|-------------|
| R-047 | MUST | Embedded artifact `type` field MUST be `"embedded"` |
| R-048 | MUST | Embedded artifact `path` field is required |
| R-049 | MUST | Embedded artifact `digest` field is required, format `sha256:{hex}` |
| R-050 | MUST | Embedded artifact `size` field is required |
| R-312 | MUST | Embedded artifact `path` values MUST be unique (codepoint-for-codepoint) |

### Referenced Artifacts

> **Note:** Referenced artifact support is OPTIONAL. Implementations MAY choose to support only embedded artifacts.

| ID | Level | Requirement |
|----|-------|-------------|
| R-051 | MAY | Implementations MAY support referenced artifacts; if supported, `type` field MUST be `"reference"` |
| R-052 | MAY | If referenced artifacts are supported, `name` field is required |
| R-053 | MAY | If referenced artifacts are supported, `uri` field is required |
| R-054 | MAY | If referenced artifacts are supported, `access` field with `policy` is required |
| R-055 | MUST | If `digest` is present and bytes are fetched, clients MUST validate that fetched bytes match the digest |
| R-056 | MUST | Digest mismatch MUST be reported as verification failure |
| R-057 | MUST | If `digest` is absent, clients MUST clearly indicate reference was not verified |
| R-058 | SHOULD | Publishers who can provide stable bytes SHOULD include a digest |

### External URI Handling

> **Note:** This section applies only to implementations that support referenced artifacts (see Section 4.3 of pack.md).

| ID | Level | Requirement |
|----|-------|-------------|
| R-059 | MUST NOT | Clients MUST NOT automatically fetch external URIs from referenced artifacts |
| R-060 | MUST | Explicit user approval or policy allowlist MUST be required for external fetches |
| R-061 | MUST | External URI scheme MUST be `https` |
| R-062 | MUST | URIs with userinfo or fragments MUST be rejected |
| R-063 | SHOULD | Clients SHOULD resolve host and reject loopback, link-local, and private network ranges |
| R-328 | SHOULD | IP blocklist checks SHOULD normalize IP to canonical form before comparison |
| R-329 | SHOULD | Implementations SHOULD handle obfuscated IP forms (decimal, hex, octal, IPv4-mapped IPv6) |
| R-330 | SHOULD | For IDN hostnames, implementations SHOULD normalize to ASCII (Punycode) before validation |
| R-064 | SHOULD | Redirects SHOULD re-validate scheme and host restrictions before request body or credentials sent |
| R-065 | MUST NOT | Clients MUST NOT send Authorization headers or secrets to different origin than original URI |
| R-066 | MUST | Clients MUST redact access tokens or credentials from logs and error messages |

### Pack Digest Computation

| ID | Level | Requirement |
|----|-------|-------------|
| R-067 | MUST | For each embedded artifact, verify `SHA256(bytes)` matches `artifact.digest` |
| R-068 | MUST | Build canonical artifact list as UTF-8 text: `{path}\t{digest}\n` |
| R-069 | MUST | Sort lines using byte-wise lexicographic ordering on raw UTF-8 bytes |
| R-070 | MUST NOT | Implementations MUST NOT apply locale-aware collation or Unicode normalization before sorting |
| R-071 | MUST | Each entry line ends with exactly one `\n` (0x0A) |
| R-072 | MUST NOT | No additional trailing newline or empty line permitted after final entry |
| R-073 | MUST | If embedded artifact list is empty, digest input is empty byte string |
| R-074 | MUST | Compute `SHA256` of concatenated result |
| R-075 | MUST | Format as `sha256:{hex}` (lowercase hexadecimal) |

### Pack Verification

| ID | Level | Requirement |
|----|-------|-------------|
| R-076 | MUST | Reject any top-level entry other than `manifest.json`, `artifacts/`, or `attestations/` |
| R-077 | MUST | Reject any file under `artifacts/` not listed as embedded artifact in manifest |
| R-078 | MUST | Reject any embedded artifact in manifest that does not exist in ZIP |
| R-079 | MUST | Reject duplicate ZIP entry names |
| R-080 | MUST | Reject any file under `attestations/` that is not direct child named `*.sigstore.json` |
| R-081 | MUST NOT | Implementations MUST NOT require directory entries in ZIP; only file entries are authoritative |
| R-310 | MUST | Verifiers MUST reject packs containing referenced ZIP entry names that cannot be interpreted as UTF-8 |
| R-315 | MUST | Implementations MUST treat UTF-8 decoding errors as fatal and MUST NOT replace invalid sequences |
| R-311 | MUST | `artifacts[].path` MUST match ZIP entry names as Unicode strings exactly (codepoint-for-codepoint), with no normalization or case folding |
| R-313 | MUST | Verifiers MUST reject ZIP file entries not referenced by the manifest, except `manifest.json` and files under `attestations/` |
| R-082 | MUST | All path comparisons MUST be codepoint-for-codepoint on Unicode strings after UTF-8 decoding |
| R-083 | MUST NOT | No percent-decoding, Unicode normalization, or case folding permitted |
| R-084 | MUST NOT | Implementations MUST NOT apply case folding during path comparison |
| R-085 | SHOULD | On case-insensitive filesystems, implementations SHOULD warn if pack contains colliding paths |
| R-086 | MUST | Implementations MUST reject packs containing Windows reserved device names on all platforms |

### Security

| ID | Level | Requirement |
|----|-------|-------------|
| R-087 | MUST | Reject paths containing `..` segments |
| R-088 | MUST | Reject paths containing backslashes |
| R-089 | MUST | Reject absolute paths (starting with `/`) |
| R-090 | MUST | Validate paths match allowed character set |
| R-091 | MUST | Implementations MUST enforce size limits for maximum artifact size, pack size, and artifact count |
| R-092 | MUST | Implementations MUST track total uncompressed bytes during extraction |
| R-093 | MUST | Implementations MUST abort immediately when any limit is exceeded |
| R-094 | MUST NOT | Implementations MUST NOT buffer entire uncompressed content before checking limits |
| R-095 | MUST | Streaming extraction with byte counting is required |
| R-324 | MUST | Extraction byte accounting MUST use delta bytes, not cumulative bytes |
| R-325 | MUST | If ZIP library reports cumulative bytes, implementation MUST convert to delta |
| R-096 | MUST | Implementations MUST enforce minimums and MUST NOT allow limits to be disabled entirely |
| R-097 | MUST | Reject archive entries that are symlinks or device files |
| R-326 | MUST NOT | Implementations MUST NOT attempt to detect hard links via ZIP metadata (unreliable) |
| R-327 | MUST | Implementations MUST extract all file entries as regular files (never create actual hard links) |
| R-098 | MUST | Check entry type before extraction begins, not after files written |
| R-099 | MUST NOT | Implementations MUST NOT follow symbolic links when writing extracted content |
| R-100 | SHOULD | Implementations SHOULD use `O_NOFOLLOW` or equivalent to prevent symlink following |
| R-101 | SHOULD | Implementations SHOULD reject AppleDouble and resource fork entries (`__MACOSX/`, `._` prefixed files) |
| R-102 | MUST | Implementations MUST reject archives with multiple entries for same path |
| R-103 | SHOULD | Implementations SHOULD extract to temporary directory and atomically rename after validation |
| R-104 | SHOULD | Implementations SHOULD ignore ZIP permission bits and apply safe defaults |
| R-105 | MUST NOT | Implementations MUST NOT log access tokens, Authorization headers, or credentials |
| R-106 | MUST | Implementations MUST redact sensitive query parameters and headers in logs/error messages |

### Attestations in Packs

| ID | Level | Requirement |
|----|-------|-------------|
| R-107 | MUST | Attestation files MUST be placed in `attestations/` directory |
| R-108 | MUST | Single-origin packs: filename MUST be `{key_id}.sigstore.json` |
| R-109 | MUST | Multi-origin packs: ALL attestation filenames MUST include origin prefix |
| R-110 | MUST | Multi-origin format: `{origin_host}@{key_id}.sigstore.json` |
| R-111 | MUST | Each attestation MUST be in a separate file |
| R-112 | MUST | `key_id` values MUST satisfy Key Identifier Requirements in registry.md §4.3.4 |
| R-113 | MUST | Verifiers MUST recompute `manifest_digest` from `manifest.json` using JCS (RFC 8785) |
| R-114 | MUST | Verifiers MUST verify pack's `pack_digest` per Section 5.2 |
| R-115 | MUST | Verifiers MUST verify attestation signature per attestation.md Section 5.1 |
| R-116 | MUST | Conforming implementations MUST pass test vectors in `/test-vectors/` directory |

---

## Attestation Format (attestation.md)

### Sigstore Bundle

| ID | Level | Requirement |
|----|-------|-------------|
| R-117 | MUST | Attestations MUST use Sigstore bundle format |
| R-118 | MUST | Bundle `mediaType` MUST be `"application/vnd.dev.sigstore.bundle.v0.3+json"` or later compatible version |
| R-119 | MUST | Bundle MUST include `verificationMaterial` with certificate chain and transparency log entry |
| R-120 | MUST | Bundle MUST include `dsseEnvelope` with signature |
| R-121 | MUST | All Base64 fields MUST use RFC 4648 standard Base64 with padding |
| R-122 | MUST NOT | URL-safe Base64 variants MUST NOT be used |
| R-331 | SHOULD | When decoding Base64, implementations SHOULD accept padded and unpadded forms |
| R-348 | MAY | Implementations MAY ignore ASCII whitespace (SP, HT, CR, LF) in Base64 input |
| R-332 | MUST | When encoding Base64 output, implementations MUST produce canonical RFC 4648 with padding |
| R-333 | MUST | Implementations MUST reject Base64 input containing non-Base64, non-whitespace characters |
| R-349 | MUST | URL-safe Base64 alphabet (`-` and `_`) MUST be rejected in Base64 input |

### Verification Material

| ID | Level | Requirement |
|----|-------|-------------|
| R-350 | MUST | `verificationMaterial` MUST include certificate chain (`x509CertificateChain`) |
| R-351 | MUST | `verificationMaterial` MUST include transparency log entries (`tlogEntries`) |
| R-352 | MUST | Certificate chain MUST be verified against Sigstore trusted root (Fulcio CA) |
| R-353 | MUST | Transparency log entry MUST include inclusion proof |
| R-354 | MUST | Inclusion proof MUST be verified against Rekor public key |

### DSSE Envelope

| ID | Level | Requirement |
|----|-------|-------------|
| R-355 | MUST | Attestations MUST use DSSE (Dead Simple Signing Envelope) format |
| R-356 | MUST | `payloadType` is required and MUST be `"application/vnd.in-toto+json"` |
| R-357 | MUST | `payload` is required and MUST be Base64-encoded in-toto Statement |
| R-123 | MUST | Attestations MUST be valid JSON documents encoded in UTF-8 |

### in-toto Statement

| ID | Level | Requirement |
|----|-------|-------------|
| R-358 | MUST | Statement `_type` is required and MUST be `"https://in-toto.io/Statement/v1"` |
| R-359 | MUST | `subject` array is required |
| R-360 | MUST | `subject` array MUST contain exactly one entry |
| R-361 | MUST | Subject `name` is required and MUST be `"manifest.json"` |
| R-362 | MUST | Subject `digest` object is required with `sha256` field |
| R-363 | MUST | Subject `digest.sha256` MUST be 64 lowercase hexadecimal characters (no prefix) |
| R-364 | MUST | `predicateType` is required and MUST be `"https://evidencepack.org/attestation/v1"` |
| R-365 | MUST | `predicate` object is required and MUST be an empty object `{}` |

### Manifest Digest Computation

| ID | Level | Requirement |
|----|-------|-------------|
| R-143 | MUST | Parse `manifest.json` as JSON |
| R-144 | MUST | Apply JCS canonicalization (RFC 8785) |
| R-145 | MUST | Compute SHA-256 of canonical UTF-8 bytes |
| R-146 | MUST | Express as 64 lowercase hex characters (no `sha256:` prefix for in-toto digest) |
| R-147 | MUST | Signing MUST fail if `manifest.json` cannot be parsed or canonicalized |
| R-137 | MUST | Implementations MUST produce canonical bytes exactly as RFC 8785 defines |
| R-138 | MUST | Canonicalization MUST reject non-finite numbers |
| R-139 | MUST | `NaN`, `Infinity`, and `-Infinity` MUST be treated as invalid JSON values |
| R-334 | MUST | Implementations MUST reject JSON documents containing duplicate keys at any nesting level |
| R-335 | MUST | Duplicate key detection MUST occur during or before parsing, not after |

### Verification Process

| ID | Level | Requirement |
|----|-------|-------------|
| R-153 | MUST | Parse Sigstore bundle JSON |
| R-366 | MUST | Verify bundle `mediaType` is valid Sigstore bundle media type |
| R-367 | MUST | Verify certificate chain against Sigstore trusted root (Fulcio CA) |
| R-368 | MUST | Verify transparency log inclusion proof against Rekor |
| R-369 | MUST | Verify signature over DSSE envelope |
| R-370 | MUST | Base64-decode `payload` to get in-toto Statement bytes |
| R-371 | MUST | Parse and validate statement structure |
| R-372 | MUST | Validate `_type` is `"https://in-toto.io/Statement/v1"` |
| R-373 | MUST | Validate `predicateType` is `"https://evidencepack.org/attestation/v1"` |
| R-374 | MUST | Validate `subject` contains exactly one entry with `name: "manifest.json"` |
| R-154 | MUST | Recompute manifest digest from `manifest.json` using JCS |
| R-375 | MUST | Compare to `subject[0].digest.sha256`; fail if mismatch |
| R-155 | MUST | Verify pack `pack_digest` per pack.md Section 5.2 |

### Identity Verification

| ID | Level | Requirement |
|----|-------|-------------|
| R-160 | MUST | If expected identity configured, verify certificate identity matches |
| R-161 | SHOULD | Verifiers SHOULD configure expected signer identities |
| R-162 | MUST | Verification MUST fail if expected identity configured and certificate does not match |

### Verification Failures

| ID | Level | Requirement |
|----|-------|-------------|
| R-376 | MUST | Verification MUST fail if Sigstore bundle is malformed |
| R-377 | MUST | Verification MUST fail if `mediaType` is not a valid Sigstore bundle media type |
| R-378 | MUST | Verification MUST fail if `verificationMaterial` is missing |
| R-379 | MUST | Verification MUST fail if `dsseEnvelope` is missing |
| R-380 | MUST | Verification MUST fail if certificate chain is invalid or untrusted |
| R-381 | MUST | Verification MUST fail if transparency log proof is invalid |
| R-382 | MUST | Verification MUST fail if signature verification fails |
| R-383 | MUST | Verification MUST fail if `payloadType` is not `"application/vnd.in-toto+json"` |
| R-384 | MUST | Verification MUST fail if `payload` is not valid Base64 |
| R-385 | MUST | Verification MUST fail if decoded statement is not valid JSON |
| R-386 | MUST | Verification MUST fail if `_type` is not `"https://in-toto.io/Statement/v1"` |
| R-387 | MUST | Verification MUST fail if `predicateType` is not `"https://evidencepack.org/attestation/v1"` |
| R-388 | MUST | Verification MUST fail if `subject` does not contain exactly one entry |
| R-389 | MUST | Verification MUST fail if `subject[0].name` is not `"manifest.json"` |
| R-171 | MUST | Verification MUST fail if manifest digest does not match `subject[0].digest.sha256` |
| R-172 | MUST | Verification MUST fail if `pack_digest` does not match recomputed value |
| R-180 | MUST | Verification MUST fail if `manifest.json` cannot be parsed or canonicalized |
| R-390 | MUST | Verification MUST fail if expected identity configured and certificate identity does not match |

### Security

| ID | Level | Requirement |
|----|-------|-------------|
| R-190 | MUST | Verifiers MUST explicitly configure expected signer identities for production use |
| R-191 | MUST | Valid signature alone does not establish trust; identity verification is required |
| R-192 | MUST | Conforming implementations MUST pass attestation test vectors |

---

## Summary Statistics

| Specification | MUST | MUST NOT | SHOULD | MAY | Total |
|---------------|------|----------|--------|-----|-------|
| pack.md | 101 | 30 | 16 | 5 | 152 |
| attestation.md | 58 | 2 | 3 | 1 | 64 |
| **TOTAL** | **159** | **32** | **19** | **6** | **216** |

> **Note:** Referenced artifact requirements (R-051–R-054, R-063, R-064, R-328–R-330) are conditional on implementations choosing to support referenced artifacts.

**Note:** Requirements are numbered with gaps from removed or renumbered rules during specification evolution.

---

## Cross-References

| Topic | Rules |
|-------|-------|
| Sigstore Bundle | R-117-120, R-350-354, R-366-369, R-376-382 |
| DSSE Envelope | R-355-357, R-383-385 |
| in-toto Statement | R-358-365, R-370-374 |
| Identity Verification | R-160-162, R-390 |
| Timestamps | R-034, R-035 |
| Digests (SHA-256) | R-037, R-049, R-067-075, R-143-146, R-362-363 |
| Base64 Encoding | R-121-122, R-331-333, R-348-349, R-357, R-384 |
| JCS Canonicalization | R-137-139, R-144, R-147, R-319-320, R-334-335 |
| Path Validation | R-009-022, R-082-090, R-310-313, R-315, R-317-318, R-321-323, R-345-347 |
| URL Safety | R-059-066, R-328-330 |
| Size Limits | R-091-096, R-324-325 |
| Logging/Secrets | R-066, R-105-106 |
| Semantic Extensibility | R-028a-028f |
