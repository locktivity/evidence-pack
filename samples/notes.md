# Sample Evidence Packs

This directory contains narrative sample packs demonstrating the Evidence Pack format evolution over time.

## Scenario: ACME Corp Quarterly Evidence Collection

ACME Corp collects quarterly security evidence for compliance. Each version represents a quarterly release:

- **v1.0.0** (Jan 7, 2026): Initial evidence collection — GitHub only
- **v1.1.0** (Jan 14, 2026): Added AWS evidence, improved GitHub metrics
- **v1.2.0** (Jan 21, 2026): Added Okta evidence, expanded AWS and GitHub coverage

## Using These Samples

Each version directory contains:
- `manifest.json` — Reference manifest showing expected structure (epbuild generates its own)
- `artifacts/` — The evidence artifacts

### Building a Pack

Use `epbuild` to create a ZIP archive from a sample directory:

```bash
epbuild --stream acme-corp/prod --in samples/v1.0.0/artifacts --out samples/v1.0.0.zip
```

### Comparing Versions

Use `epdiff` to see changes between versions:

```bash
epdiff samples/v1.0.0.zip samples/v1.1.0.zip
# Shows added, removed, and modified artifacts
```

### Verifying Samples

```bash
epverify samples/v1.0.0.zip
# Validates manifest, digests, and structure
```

## Evidence Evolution

The sample packs demonstrate a realistic evidence lifecycle with improving security posture:

1. **v1.0.0**: Baseline — GitHub org settings and branch protection (66.7% coverage)
2. **v1.1.0**: Expanded — Added AWS identity posture, improved branch protection (81.8%)
3. **v1.2.0**: Mature — Added Okta MFA, AWS storage, GitHub security features (92% branch protection)

This mirrors real-world evidence management where:
- Evidence is collected periodically
- Coverage expands as integrations are added
- Security posture improves over time
- Metrics are tracked across collections
