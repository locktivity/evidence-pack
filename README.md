<p align="center">
  <img src="assets/evidence-pack-lockup-dark.png" alt="Evidence Pack" width="400">
</p>

# Evidence Packs

Evidence Packs define a ZIP-based format for packaging security evidence with explicit metadata, integrity checks, and optional signatures.

This repository contains:

- the **v1 specification** (format, Sigstore attestations, profiles)
- **conformance test vectors** (the interoperability contract)
- **sample packs** (realistic examples)

Reference tooling (non-normative) lives in:

- **Go CLI & SDK (`epack`):** [github.com/locktivity/epack](https://github.com/locktivity/epack)

## Why Evidence Packs?

Security evidence typically lives in portals, PDFs, and one-off exports. Evidence Packs treat evidence more like software releases:

| Property | Description |
|----------|-------------|
| **Portable** | Delivered as a ZIP archive with a fixed structure |
| **Verifiable** | Integrity via digests; authenticity when attestations are validated against a trust policy (e.g., trusted keys) |
| **Versioned** | Declared format version plus publisher-defined release identifiers |
| **Diffable** | Artifact index enables deterministic comparison (path + digest; canonical manifest rules) |

## Specification

The normative spec lives in `spec/v1/`:

| Document | Description |
|----------|-------------|
| [spec/v1/pack.md](spec/v1/pack.md) | Pack format and digest rules |
| [spec/v1/attestation.md](spec/v1/attestation.md) | Sigstore signing and verification |
| [spec/v1/rules.md](spec/v1/rules.md) | Validation rules and error handling |
| [spec/v1/profiles.md](spec/v1/profiles.md) | Profiles & semantic validation (Draft) |
| [spec/v1/security-appendix.md](spec/v1/security-appendix.md) | Security considerations |

## Conformance

Conformance is defined by:

1. The **normative requirements** in the spec (`MUST`/`SHOULD` language), and
2. The **test vectors** in `test-vectors/`

A verifier implementation is considered conformant if it matches the expected results for all applicable vectors.

See [CONFORMANCE.md](CONFORMANCE.md).

## Repository layout

```
spec/
  v1/
    pack.md              # Pack format and digest rules
    attestation.md       # Sigstore signing and verification
    rules.md             # Validation rules and error handling
    profiles.md          # Profiles & semantic validation
    security-appendix.md # Security considerations

test-vectors/
  pack-digest/           # Pack digest computation vectors
  manifest-digest/       # Manifest digest (JCS) vectors
  path-validation/       # Path validation vectors
  zip-safety/            # Archive safety vectors
  structure/             # Manifest structure vectors
  limits/                # Size and count limit vectors
  jcs/                   # JSON Canonicalization vectors
  manifest/              # Manifest parsing vectors
  attestation/           # Attestation verification vectors

samples/
  v1.0.0/
  v1.1.0/
  v1.2.0/

profiles/                # Profile definitions
```

## Design goals

- **Interop first:** vectors are the contract; implementations may differ.
- **Boring security:** deterministic rules, explicit error cases, safe archive handling.
- **Minimal v1:** keep the core stable; extensions go behind namespaced fields.

## Security

Security issues in the spec, vectors, or samples should be reported privately. See [SECURITY.md](SECURITY.md).

## Governance and versioning

This spec is developed in the open. Versioning rules and change process are in [GOVERNANCE.md](GOVERNANCE.md).

## License

Apache 2.0 — see [LICENSE](LICENSE).

## Trademarks

"Locktivity" and related marks are trademarks of Locktivity. This repository contains an open specification; implementations may claim conformance if they meet the conformance requirements. See [TRADEMARKS.md](TRADEMARKS.md).

## Maintainers

Maintained by [Locktivity](https://github.com/locktivity) in the open.

---

<p align="center">
  <img src="assets/locktivity-icon.png" alt="Locktivity" width="40" />
</p>
<p align="center">
  <strong>Built by Locktivity</strong><br/>
  <a href="https://locktivity.com">Locktivity</a> is developing Evidence Packs in the open because we believe portable, verifiable security evidence is a problem bigger than any one vendor.
</p>
