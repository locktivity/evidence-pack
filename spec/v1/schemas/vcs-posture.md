# Version Control Posture Schema

**Type Identifier:** `evidencepack/vcs-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for version control system security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (GitHub, GitLab, Bitbucket, etc.).

Whether the organization requires two-factor authentication is `sign_in.mfa_required` in [access-posture.md](access-posture.md). Dependabot, secret scanning, and code scanning coverage are in [vuln-scanning.md](vuln-scanning.md).

Only `schema_version`, `collected_at`, `provider`, and `organization` are required. A value the collector could not read is absent or `null`.

Each percentage has a count beside it. `branch_protection.pr_required_pct` is `pr_required_repository_count` as a share of `scanned_repository_count`, and the other branch protection percentages follow the same pattern. `repo_coverage_pct` is `scanned_repository_count` as a share of `repository_count`. Percentages are whole numbers. A percentage where more is better rounds down, so 47 of 48 repositories reads 97. The counts are optional. When they are absent, `repositories_present` shows whether the scan covered any repositories.

An artifact is collected at the trust, audit, or internal level. The fields whose names end in `_count` are written at the audit and internal levels only. Every other field is written at every level, including `repositories_present` and the percentages. The level is not a field of the artifact. The example in Section 3 is an audit-level artifact.

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider", "organization"],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "1.0.0"
    },
    "collected_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp when data was collected"
    },
    "provider": {
      "type": "string",
      "enum": ["github", "gitlab", "bitbucket", "azure_devops", "other"],
      "description": "Version control provider"
    },
    "organization": {
      "type": "string",
      "description": "Organization, group, or workspace name"
    },
    "repositories_present": {
      "type": ["boolean", "null"],
      "description": "Whether the scan covered any repositories. When false, the branch protection percentages are null"
    },
    "repository_count": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Repositories in the organization that the include and exclude patterns could select"
    },
    "scanned_repository_count": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Repositories the scan covered. The branch protection percentages count these"
    },
    "repo_coverage_pct": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "description": "Percentage of repositories covered by the scan (based on include/exclude patterns)"
    },
    "branch_protection": {
      "type": ["object", "null"],
      "description": "Coverage percentages for branch protection rules on default branches",
      "properties": {
        "pr_required_repository_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Scanned repositories requiring pull requests before merge"
        },
        "approving_reviews_repository_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Scanned repositories requiring at least one approving review"
        },
        "status_checks_repository_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Scanned repositories requiring status checks to pass"
        },
        "signed_commits_repository_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Scanned repositories requiring signed commits"
        },
        "pr_required_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring pull requests before merge"
        },
        "approving_reviews_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring at least one approving review"
        },
        "status_checks_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring status checks to pass"
        },
        "signed_commits_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring signed commits"
        }
      }
    },
    "org_security": {
      "type": ["object", "null"],
      "deprecated": true,
      "description": "Kept for profiles written before epack v0.10.0, and to be removed in a later release. These values are in evidencepack/access-posture@v1 now",
      "properties": {
        "two_factor_required": {
          "type": ["boolean", "null"],
          "deprecated": true,
          "description": "Whether organization requires 2FA for all members. The same value is sign_in.mfa_required in evidencepack/access-posture@v1"
        }
      }
    },
    "security_features": {
      "type": ["object", "null"],
      "deprecated": true,
      "description": "Coverage percentages for security scanning features. Kept for profiles written before epack v0.10.0, and to be removed in a later release. These values are in evidencepack/vuln-scanning@v1 now",
      "properties": {
        "vuln_alerts_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "deprecated": true,
          "description": "Percentage of repos with dependency vulnerability alerts enabled. In evidencepack/vuln-scanning@v1, the same value is the coverage_pct of the dependency scanner's entry in scanners"
        },
        "secret_scanning_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "deprecated": true,
          "description": "Percentage of repos with secret scanning enabled. In evidencepack/vuln-scanning@v1, the same value is the coverage_pct of the secret scanner's entry in scanners"
        },
        "code_scanning_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "deprecated": true,
          "description": "Percentage of repos with code scanning (SAST) enabled. In evidencepack/vuln-scanning@v1, the same value is the coverage_pct of the code scanner's entry in scanners"
        }
      }
    }
  }
}
```

## 3. Example

```json
{
  "schema_version": "1.0.0",
  "collected_at": "2026-03-18T10:00:00Z",
  "provider": "github",
  "organization": "acme-corp",
  "repositories_present": true,
  "repository_count": 50,
  "scanned_repository_count": 48,
  "repo_coverage_pct": 96,
  "branch_protection": {
    "pr_required_repository_count": 44,
    "approving_reviews_repository_count": 41,
    "status_checks_repository_count": 39,
    "signed_commits_repository_count": 5,
    "pr_required_pct": 91,
    "approving_reviews_pct": 85,
    "status_checks_pct": 81,
    "signed_commits_pct": 10
  }
}
```

## 4. Field Mapping by Provider

| Normalized Field | GitHub | GitLab |
|-----------------|--------|--------|
| `repo_coverage_pct` | Repositories matched by include/exclude patterns | Projects matched by include/exclude patterns |
| `branch_protection.pr_required_pct` | Branch protection or ruleset: require a pull request | Protected default branch with no direct push access |
| `branch_protection.approving_reviews_pct` | Require approving reviews | Approvals required before merge |
| `branch_protection.status_checks_pct` | Require status checks | Pipelines must succeed |
| `branch_protection.signed_commits_pct` | Require signed commits | Not collected |

Each `_repository_count` field comes from the same setting as its percentage.

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Change management | `branch_protection.pr_required_pct` | `>= 90` |
| Code review required | `branch_protection.approving_reviews_pct` | `>= 90` |
| CI/CD enforcement | `branch_protection.status_checks_pct` | `>= 80` |

Two-factor authentication is mapped in [access-posture.md](access-posture.md) Section 5. Dependency, secret, and code scanning are mapped in [vuln-scanning.md](vuln-scanning.md) Section 5.

## 6. Repository Coverage

`repository_count` is the number of repositories the include and exclude patterns could select, and `scanned_repository_count` is the number the scan covered. `repo_coverage_pct` is the second as a share of the first. When `repositories_present` is false, the scan covered no repositories and the branch protection percentages are `null`.

Collectors SHOULD document their include/exclude patterns and explain why certain repositories may be excluded (e.g., archived repos, forks, specific patterns).

## 7. Deprecated Fields

epack v0.10.0 moved `org_security` and `security_features` to [access-posture.md](access-posture.md) and [vuln-scanning.md](vuln-scanning.md). They are deprecated here, and producers write them while profiles move to the new fields. They are removed in a later revision of the v1 draft.

| Deprecated field | Replacement |
|-----------------|-------------|
| `org_security.two_factor_required` | access-posture `sign_in.mfa_required` |
| `security_features.vuln_alerts_pct` | vuln-scanning `coverage_pct` of the dependency scanner's entry in `scanners` |
| `security_features.secret_scanning_pct` | vuln-scanning `coverage_pct` of the secret scanner's entry in `scanners` |
| `security_features.code_scanning_pct` | vuln-scanning `coverage_pct` of the code scanner's entry in `scanners` |
