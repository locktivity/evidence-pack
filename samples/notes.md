# Sample Evidence Packs

This directory contains narrative sample packs demonstrating the Evidence Pack format evolution over time.

## Scenario: ACME Corp Quarterly Evidence Collection

ACME Corp collects quarterly security evidence for compliance. Each version represents a quarterly release:

- **v1.0.0** (Jan 7, 2026): Initial evidence collection, GitHub only
- **v1.1.0** (Jan 14, 2026): Added AWS evidence, improved GitHub metrics
- **v1.2.0** (Jan 21, 2026): Added Okta evidence, expanded AWS and GitHub coverage

## Using These Samples

Each version directory contains:
- `manifest.json`, a reference manifest. A pack built from the sample gets its own manifest
- `artifacts/`, the evidence artifacts

### Building a Pack

Use `epack build` from the repository root to create a pack from a sample directory. With `--file`, each artifact keeps its path from the reference manifest:

```bash
epack build samples/v1.0.0.epack --stream acme-corp/prod \
  --file samples/v1.0.0/artifacts/github/org-settings.json:artifacts/github/org-settings.json \
  --file samples/v1.0.0/artifacts/github/branch-protection.json:artifacts/github/branch-protection.json
```

### Comparing Versions

Use `epack diff` to list the artifacts added, removed, and changed between versions:

```bash
epack diff samples/v1.0.0.epack samples/v1.1.0.epack
```

### Verifying Samples

The samples are unsigned, so verify their digests and structure only:

```bash
epack verify --integrity-only samples/v1.0.0.epack
```

## Evidence Evolution

The sample packs demonstrate a realistic evidence lifecycle with improving security posture:

1. **v1.0.0**: Baseline. GitHub org settings and branch protection (66.7% coverage)
2. **v1.1.0**: Expanded. Added AWS identity posture, improved branch protection (81.8%)
3. **v1.2.0**: Mature. Added Okta MFA, AWS storage, GitHub security features (92% branch protection)

This mirrors real-world evidence management where:
- Evidence is collected periodically
- Coverage expands as integrations are added
- Security posture improves over time
- Metrics are tracked across collections
