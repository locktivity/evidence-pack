# Attestation Format

**Version:** 1.0
**Status:** Draft

## 1. Introduction

Evidence Pack attestations use [Sigstore](https://sigstore.dev) for cryptographic signing and verification. Sigstore provides keyless signing via OIDC identity, transparency logging, and self-contained verification bundles.

This document defines how Sigstore bundles are used within Evidence Packs.

## 2. Overview

### 2.1 What Sigstore Provides

- **Keyless signing:** Signers authenticate via OIDC (GitHub, Google, Microsoft, etc.) and receive short-lived certificates from Fulcio
- **Transparency log:** All signatures are recorded in Rekor, a public append-only log
- **Self-contained bundles:** `.sigstore.json` files include all verification material (certificates, inclusion proofs, timestamps)
- **Offline verification:** Bundles contain stapled proofs that enable verification without network access

### 2.2 Attestation Structure

Each attestation is a Sigstore bundle containing:

1. **DSSE Envelope:** Contains the signature over an in-toto statement
2. **Verification Material:** Certificate chain and transparency log proof
3. **in-toto Statement:** Describes what is being attested (the manifest digest)

## 3. Sigstore Bundle Format

Attestation files use the Sigstore bundle format with media type `application/vnd.dev.sigstore.bundle.v0.3+json`.

### 3.1 Bundle Structure

```json
{
  "mediaType": "application/vnd.dev.sigstore.bundle.v0.3+json",
  "verificationMaterial": {
    "x509CertificateChain": {
      "certificates": [
        { "rawBytes": "<base64-encoded-certificate>" }
      ]
    },
    "tlogEntries": [
      {
        "logIndex": "123456",
        "logId": { "keyId": "<base64>" },
        "kindVersion": { "kind": "dsse", "version": "0.0.1" },
        "integratedTime": "1234567890",
        "inclusionPromise": { "signedEntryTimestamp": "<base64>" },
        "inclusionProof": {
          "logIndex": "123456",
          "rootHash": "<base64>",
          "treeSize": "1000000",
          "hashes": ["<base64>", "..."]
        },
        "canonicalizedBody": "<base64>"
      }
    ]
  },
  "dsseEnvelope": {
    "payloadType": "application/vnd.in-toto+json",
    "payload": "<base64(in-toto statement)>",
    "signatures": [
      {
        "keyid": "",
        "sig": "<base64-signature>"
      }
    ]
  }
}
```

### 3.2 Verification Material

The `verificationMaterial` contains everything needed for offline verification:

| Field | Description |
|-------|-------------|
| `x509CertificateChain` | Fulcio-issued certificate chain binding identity to ephemeral key |
| `tlogEntries` | Rekor transparency log entry with inclusion proof |
| `inclusionProof` | Merkle proof that signature was logged |

### 3.3 DSSE Envelope

The `dsseEnvelope` follows the [DSSE specification](https://github.com/secure-systems-lab/dsse):

| Field | Description |
|-------|-------------|
| `payloadType` | MUST be `"application/vnd.in-toto+json"` |
| `payload` | Base64-encoded in-toto statement |
| `signatures` | Array containing exactly one signature |

## 4. in-toto Statement

The DSSE payload (when Base64-decoded) is an in-toto Statement:

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    {
      "name": "manifest.json",
      "digest": {
        "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
      }
    }
  ],
  "predicateType": "https://evidencepack.org/attestation/v1",
  "predicate": {}
}
```

### 4.1 Statement Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `_type` | String | Yes | MUST be `"https://in-toto.io/Statement/v1"` |
| `subject` | Array | Yes | Array containing exactly one subject |
| `predicateType` | String | Yes | MUST be `"https://evidencepack.org/attestation/v1"` |
| `predicate` | Object | Yes | MUST be an empty object `{}` |

### 4.2 Subject

The `subject` array MUST contain exactly one entry describing the manifest being attested:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | String | Yes | MUST be `"manifest.json"` |
| `digest` | Object | Yes | Object with `sha256` field containing manifest digest |

The `digest.sha256` field contains the SHA-256 hash of the manifest's canonical form (64 lowercase hexadecimal characters, no `sha256:` prefix).

### 4.3 Predicate

The predicate MUST be an empty object `{}`.

All relevant data (stream, artifacts, pack_digest) is already in the manifest, which is cryptographically bound via the subject digest. Including this data in the predicate would be redundant.

## 5. Manifest Digest Computation

The manifest digest (used in `subject[0].digest.sha256`) is computed as follows:

1. Parse `manifest.json` as JSON
2. Apply JCS canonicalization (RFC 8785)
3. Compute SHA-256 of the canonical UTF-8 bytes
4. Express as 64 lowercase hexadecimal characters (no prefix)

Signing MUST fail if `manifest.json` cannot be parsed as JSON or canonicalized.

**JSON numbers:** Canonicalization MUST reject non-finite numbers. `NaN`, `Infinity`, and `-Infinity` MUST be treated as invalid JSON values.

**Duplicate keys:** Implementations MUST reject JSON documents containing duplicate keys at any nesting level.

## 6. Signing Process

### 6.1 Using cosign

The recommended signing method uses cosign's blob signing:

```bash
# Extract manifest from pack
unzip -p pack.zip manifest.json > manifest.json

# Sign with Sigstore (keyless)
cosign sign-blob manifest.json \
  --bundle attestations/signer.sigstore.json \
  --yes

# Add attestation to pack
zip pack.zip attestations/signer.sigstore.json
```

### 6.2 Using sigstore-go

For programmatic signing:

```go
import (
    "github.com/sigstore/sigstore-go/pkg/sign"
)

// Create signer with OIDC identity
signer, err := sign.NewSigner(ctx, sign.WithFulcio(), sign.WithRekor())

// Sign manifest bytes
bundle, err := signer.SignBlob(ctx, manifestBytes)

// Write bundle to attestations/
```

### 6.3 Attestation with Custom Predicate

To include the Evidence Pack predicate:

```bash
# Create in-toto statement
cat > statement.json << EOF
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{"name": "manifest.json", "digest": {"sha256": "..."}}],
  "predicateType": "https://evidencepack.org/attestation/v1",
  "predicate": {}
}
EOF

# Sign statement as attestation
cosign attest-blob \
  --predicate statement.json \
  --bundle attestations/signer.sigstore.json \
  manifest.json
```

## 7. Verification Process

### 7.1 Steps

**Inputs:** Evidence Pack (ZIP), expected signer identities (optional)

**Output:** Verification result (valid, identity info) or error

1. Extract `manifest.json` from the pack
2. Compute the manifest digest per Section 5
3. For each `attestations/*.sigstore.json`:
   a. Parse the Sigstore bundle
   b. Verify the bundle against the Sigstore trusted root (Fulcio CA, Rekor log)
   c. Verify the signature over the DSSE envelope
   d. Base64-decode the payload to get the in-toto statement
   e. Validate statement structure:
      - `_type` MUST be `"https://in-toto.io/Statement/v1"`
      - `predicateType` MUST be `"https://evidencepack.org/attestation/v1"`
      - `subject[0].name` MUST be `"manifest.json"`
   f. Compare `subject[0].digest.sha256` to computed manifest digest
   g. If expected identities provided, verify certificate identity matches
4. Return verification result with signer identities

### 7.2 Identity Verification

Sigstore certificates contain the signer's identity from OIDC. Common identity types:

| OIDC Provider | Identity Example |
|---------------|------------------|
| GitHub Actions | `https://github.com/org/repo/.github/workflows/release.yaml@refs/heads/main` |
| Google | `user@example.com` |
| Microsoft | `user@example.com` |
| GitHub (personal) | `user@users.noreply.github.com` |

Verifiers specify which identities to trust:

```go
policy := verify.NewPolicy(
    verify.WithArtifact(manifestBytes),
    verify.WithCertificateIdentity(
        "security@acme.com",
        "https://accounts.google.com",
    ),
)
result, err := verifier.Verify(bundle, policy)
```

### 7.3 Verification Failures

Verification MUST fail if:

- The Sigstore bundle is malformed
- The certificate chain is invalid or untrusted
- The transparency log proof is invalid
- The signature verification fails
- `payloadType` is not `"application/vnd.in-toto+json"`
- The decoded statement is not valid JSON
- `_type` is not `"https://in-toto.io/Statement/v1"`
- `predicateType` is not `"https://evidencepack.org/attestation/v1"`
- `subject[0].name` is not `"manifest.json"`
- The manifest digest does not match `subject[0].digest.sha256`
- Expected identity is specified but does not match certificate identity

## 8. Attestation Placement

### 8.1 In Evidence Packs

Attestations MUST be placed in the `attestations/` directory:

```
pack.zip
├── manifest.json
├── artifacts/
│   └── ...
└── attestations/
    └── *.sigstore.json
```

### 8.2 Filename Convention

Attestation files MUST be named using the pattern:

```
{identity-hash}.sigstore.json
```

Where `identity-hash` is the first 8 characters of the SHA-256 hash of the certificate's subject identity (email or URI).

**Example:**
```
attestations/
├── a1b2c3d4.sigstore.json    # security@acme.com
├── e5f6g7h8.sigstore.json    # auditor@thirdparty.com
└── i9j0k1l2.sigstore.json    # github.com/org/repo workflow
```

### 8.3 Multiple Attestations

A pack MAY contain multiple attestations from different signers. Each signer creates their own `.sigstore.json` bundle.

Use cases for multiple attestations:
- Vendor signs when creating the pack
- Auditor signs after reviewing
- CI/CD pipeline signs after automated checks

## 9. Offline Verification

Sigstore bundles are self-contained and support offline verification through stapled proofs:

- **Certificate validity:** The inclusion proof timestamp proves the certificate was valid at signing time
- **Transparency log:** The inclusion proof verifies the signature was logged without contacting Rekor
- **No network required:** All verification material is embedded in the bundle

For offline verification, implementations:
1. MUST use the Sigstore trusted root (Fulcio CA certificates, Rekor public key)
2. MAY cache the trusted root for air-gapped environments
3. MUST verify inclusion proofs against the stapled Rekor entry

## 10. Private Sigstore Instances

For enterprise or air-gapped environments, organizations MAY run private Sigstore infrastructure (Fulcio, Rekor, timestamp authority). Private instance configuration is out of scope for this specification version.

Future versions MAY specify:
- Trusted root configuration for private instances
- Bundle format extensions for private CA identification
- Interoperability requirements between public and private instances

## 11. Security Considerations

### 11.1 Identity Trust

Verifiers MUST explicitly configure which identities to trust. Simply verifying a valid Sigstore signature does not establish trust in the signer.

### 11.2 Transparency Log

All signatures are recorded in Rekor. This provides:
- **Non-repudiation:** Signers cannot deny signing
- **Auditability:** All signatures are publicly discoverable
- **Timestamp authority:** Inclusion time proves when signing occurred

### 11.3 Certificate Lifetime

Fulcio certificates are short-lived (typically 10 minutes). The transparency log entry's timestamp proves the signature was created during the certificate's validity period.

### 11.4 Key Compromise

Unlike traditional PKI, Sigstore's keyless model means:
- No long-lived private keys to compromise
- No key rotation required
- Identity compromise (OIDC account) is the attack vector

Organizations SHOULD:
- Use CI/CD workload identities where possible
- Enable MFA on OIDC accounts used for signing
- Monitor Rekor for unexpected signatures from their identities

## 12. Test Vectors

Conforming implementations MUST pass the attestation test vectors in `/test-vectors/`. See [pack.md](pack.md), Section 9.

Attestation-specific vectors include:
- Sigstore bundle parsing
- in-toto statement validation
- Manifest digest computation
- Identity extraction from certificates

## 13. References

- [Sigstore](https://sigstore.dev) - Keyless signing for the software supply chain
- [Fulcio](https://github.com/sigstore/fulcio) - Certificate authority for code signing
- [Rekor](https://github.com/sigstore/rekor) - Transparency log for signatures
- [sigstore-go](https://github.com/sigstore/sigstore-go) - Go client library
- [cosign](https://github.com/sigstore/cosign) - CLI for signing artifacts
- [DSSE Specification](https://github.com/secure-systems-lab/dsse) - Dead Simple Signing Envelope
- [in-toto Attestation Framework](https://github.com/in-toto/attestation) - in-toto Statement format
- [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785) - JSON Canonicalization Scheme (JCS)
