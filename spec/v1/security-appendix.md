# Security Appendix: Implementation Checklist

**Version:** 1.0
**Status:** Informative (Non-Normative)

This appendix consolidates security guidance for Evidence Pack implementers. It is derived from threat modeling and security review of the v1.0 specification. While the core specification documents contain normative requirements (MUST/SHOULD), this appendix provides additional context and a pre-release checklist.

---

## 1. Threat Model Summary

### 1.1 Attacker Profiles

| Profile | Capabilities | Goals |
|---------|--------------|-------|
| **Malicious Publisher** | Controls pack content, attestation keys | Inject malicious artifacts, exploit verifier vulnerabilities |
| **Man-in-the-Middle** | Intercepts network traffic (without TLS compromise) | Downgrade attacks, traffic analysis, cache poisoning |
| **Compromised CI** | Controls build environment, may have signing keys | Sign malicious packs, exfiltrate secrets |
| **Malicious Verifier Plugin** | Runs in verifier context | Exploit during verification, data exfiltration |
| **Confused Deputy** | Tricks verifier into unintended actions | SSRF, path traversal, resource exhaustion |

### 1.2 Highest-Impact Attack Vectors

1. **ZIP Slip / Path Traversal** — Write files outside extraction directory
2. **ZIP Bomb** — Exhaust disk/memory via compression ratio attacks
3. **SSRF via Referenced Artifacts** — Trick verifier into fetching internal URLs
4. **JCS Canonicalization Divergence** — Different implementations produce different canonical bytes
5. **TOCTOU in Extraction** — Race condition between validation and use
6. **Unicode Normalization Confusion** — Different paths that appear identical
7. **Symlink Following** — Escape extraction sandbox via symlinks
8. **Certificate Identity Confusion** — Accepting attestations from unexpected signers

---

## 2. Signer Identity Verification

### 2.1 Certificate Identity Policy

Sigstore attestations bind signatures to OIDC identities via Fulcio certificates. Verifiers SHOULD configure expected signer identities:

```
Is expected signer identity configured?
  │
  ├─ YES → Verify certificate identity matches expected
  │
  └─ NO → Is session interactive?
           │
           ├─ YES → Prompt user to approve signer identity
           │         (display certificate subject/issuer)
           │
           └─ NO → WARN that signer identity is not verified
                   (signature is valid but signer unknown)
```

### 2.2 Rationale

- **Explicit trust:** Users should know who signed the pack
- **Defense in depth:** Valid signature + trusted identity = verified attestation
- **Transparency:** Sigstore provides public audit trail via Rekor

---

## 3. Resource Limits

### 3.1 Required Limits (per pack.md Section 7.2)

| Limit | Default | Minimum | Rationale |
|-------|---------|---------|-----------|
| Max pack size | 2 GB | 10 MB | Reasonable upper bound for evidence collections |
| Max artifact size | 100 MB | 1 MB | Prevents single large files from exhausting memory |
| Max artifact count | 10,000 | 100 | Prevents file handle exhaustion |

### 3.2 Hardening Limits (per pack.md Section 7.2.2)

| Limit | Default | Rationale |
|-------|---------|-----------|
| Max compression ratio | 100:1 | Mitigates zip bomb attacks |

### 3.3 Streaming Extraction

Implementations SHOULD use streaming extraction to enforce limits during decompression rather than buffering entire archive contents in memory. Track:

- Running total of uncompressed bytes
- Per-entry compression ratio
- Entry count
- Wall-clock time elapsed

---

## 4. ZIP Safety

### 4.1 Entry Validation Checklist

Before extracting any entry:

- [ ] Path is valid UTF-8
- [ ] Path contains no NUL bytes
- [ ] Path contains no control characters (U+0000-U+001F, U+007F)
- [ ] Path contains no backslashes
- [ ] Path does not start with `/`
- [ ] Path does not contain `..` segments
- [ ] Path does not contain empty segments (`//`)
- [ ] Path is in NFC Unicode form
- [ ] Path length ≤ 240 bytes
- [ ] Each segment length ≤ 64 bytes
- [ ] Path does not use Windows reserved names
- [ ] Entry is not a symlink (check ZIP mode bits for symlink mode)
- [ ] Entry is not a device file (check ZIP mode bits for device file mode)
- [ ] Entry type matches trailing slash (directories end with `/`)
- [ ] Compressed size is reasonable (check ratio)
- [ ] Entry is not under `__MACOSX/` directory
- [ ] Entry filename does not start with `._` (AppleDouble resource fork)

### 4.2 Extraction Safety

- Check entry type before extraction begins, not after files are written
- Validate paths before any filesystem operations
- Verify destination is within extraction directory (defense in depth)
- Reject symlinks and device files based on ZIP mode bits
- Do not follow symbolic links when writing extracted content
- Use `O_NOFOLLOW` (or equivalent) when opening files to prevent symlink following
- Clean up partial extractions on failure

---

## 5. Referenced Artifact URL Safety

Referenced artifacts (`type: "reference"`) contain external URIs. If implementations fetch these URIs, they SHOULD apply URL safety checks.

### 5.1 URL Validation

1. **Scheme allowlist:** HTTPS only (reject HTTP, file://, etc.)
2. **Userinfo rejection:** No `user:pass@` in URLs
3. **Fragment rejection:** No `#fragment` in URLs

### 5.2 Fetch Policy

Implementations MUST NOT automatically fetch referenced artifact URIs. Fetching requires explicit user consent or policy allowlist.

---

## 6. JSON/JCS Safety

### 6.1 JSON Validation Checklist

- [ ] Duplicate keys are rejected at any nesting level (MUST per pack.md)
- [ ] Duplicate key detection occurs during or before parsing, not after

### 6.2 JCS Validation Checklist

- [ ] Non-finite numbers (`NaN`, `Infinity`, `-Infinity`) are rejected
- [ ] Integer overflow is handled (numbers outside safe integer range)
- [ ] Unicode surrogate pairs are handled correctly
- [ ] Lone surrogates are rejected
- [ ] String escaping matches RFC 8785 exactly
- [ ] Key sorting uses UTF-16 code unit comparison (per RFC 8785)

---

## 7. Logging and Secrets

### 7.1 What MUST NOT Be Logged

- Authorization headers
- Bearer tokens
- API keys (in headers or query parameters)
- Credentials embedded in URLs
- Private key material
- Session tokens

### 7.2 Redaction Checklist

- [ ] URL userinfo is redacted before logging
- [ ] Sensitive query parameters (`token`, `api_key`, `secret`, etc.) are redacted
- [ ] Authorization and authentication headers are redacted
- [ ] Cookie headers are redacted

---

## 8. Exit Codes

### 8.1 Structured Exit Code Ranges

| Range | Category | Example Codes |
|-------|----------|---------------|
| 0 | Success | 0 = OK |
| 1-9 | General errors | 1 = unspecified, 2 = usage error |
| 10-19 | Configuration | 10 = config not found, 11 = config parse error, 12 = trust mode required |
| 20-29 | Input/IO | 20 = file not found, 21 = read error, 22 = write error |
| 30-39 | ZIP/Archive | 30 = invalid ZIP, 31 = path traversal, 32 = zip bomb, 33 = symlink |
| 40-49 | Manifest | 40 = invalid JSON, 41 = missing field, 42 = invalid timestamp |
| 50-59 | Digest | 50 = pack digest mismatch, 51 = artifact digest mismatch |
| 60-69 | Signature | 60 = invalid signature, 61 = key not found, 62 = key revoked |
| 70-79 | Network | 70 = connection failed, 71 = TLS error, 72 = SSRF blocked |
| 80-89 | Registry | 80 = registry error, 81 = stream not found |
| 90-99 | Trust | 90 = trust not established, 91 = key pin mismatch |

---

## 9. Pre-Release Checklist

### 9.1 Security Defaults

- [ ] Trust mode requires explicit configuration (no implicit default)
- [ ] Cross-origin fetching is disabled by default
- [ ] Size limits are enforced and cannot be disabled
- [ ] All paths are validated before use

### 9.2 Resource Limits

- [ ] Pack size limit enforced (default: 2 GB)
- [ ] Artifact size limit enforced (default: 100 MB)
- [ ] Artifact count limit enforced (default: 10,000)
- [ ] Compression ratio checked (default: 100:1)
- [ ] Streaming extraction used (no buffering entire content)

### 9.3 ZIP Safety

- [ ] Path validation rejects traversal attempts
- [ ] Symlinks are rejected
- [ ] Device files are rejected
- [ ] Duplicate paths are rejected
- [ ] Unicode NFC normalization is required
- [ ] Case collisions produce warnings
- [ ] Windows reserved names are rejected on all platforms
- [ ] AppleDouble and `__MACOSX` entries are rejected

### 9.4 Network Safety

- [ ] HTTPS required (no HTTP fallback)
- [ ] TLS certificates are validated
- [ ] IP blocklist includes all private/loopback/link-local ranges
- [ ] IP obfuscation forms are handled (decimal, hex, octal)
- [ ] IPv4-mapped IPv6 addresses are normalized
- [ ] IDN hostnames are normalized to Punycode
- [ ] Redirects are limited (max 5)
- [ ] Redirects are re-validated
- [ ] Auth headers are stripped on redirect
- [ ] DNS rebinding is mitigated (resolve immediately before connect)

### 9.5 JSON and Cryptography

- [ ] Duplicate keys are rejected in all JSON documents
- [ ] Sigstore bundles verified against public trusted root
- [ ] Certificate chain validated against Fulcio CA
- [ ] Transparency log entry validated against Rekor
- [ ] JCS canonicalization matches RFC 8785
- [ ] Non-finite numbers are rejected
- [ ] Certificate identity verified against expected signer (if configured)

### 9.6 Logging

- [ ] Authorization headers are never logged
- [ ] Tokens and API keys are never logged
- [ ] URL credentials are redacted
- [ ] Sensitive query parameters are redacted
- [ ] Private key material is never logged

### 9.7 Testing

- [ ] All test vectors pass
- [ ] Differential testing against reference implementation
- [ ] Fuzz testing for ZIP parsing
- [ ] Fuzz testing for JSON/JCS parsing
- [ ] Fuzz testing for path validation

---

## 10. References

- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [RFC 8785 - JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)
- [Sigstore Documentation](https://docs.sigstore.dev/)
- [sigstore-go Library](https://github.com/sigstore/sigstore-go)
- [in-toto Attestation Framework](https://github.com/in-toto/attestation)
- [ZIP File Format Specification](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)
- [Unicode Technical Report #15 - Normalization Forms](https://unicode.org/reports/tr15/)
