# Version Control Posture Schema

**Type Identifier:** `evidencepack/vcs-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for version control system security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (GitHub, GitLab, Bitbucket, etc.).

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
    "org_security": {
      "type": "object",
      "properties": {
        "two_factor_required": {
          "type": "boolean",
          "description": "Whether organization requires 2FA for all members"
        }
      }
    },
    "repo_coverage_pct": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Percentage of repositories covered by the scan (based on include/exclude patterns)"
    },
    "branch_protection": {
      "type": "object",
      "description": "Coverage percentages for branch protection rules on default branches",
      "properties": {
        "pr_required_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring pull requests before merge"
        },
        "approving_reviews_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring at least one approving review"
        },
        "status_checks_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring status checks to pass"
        },
        "signed_commits_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos requiring signed commits"
        }
      }
    },
    "security_features": {
      "type": "object",
      "description": "Coverage percentages for security scanning features",
      "properties": {
        "vuln_alerts_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos with dependency vulnerability alerts enabled"
        },
        "secret_scanning_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos with secret scanning enabled"
        },
        "code_scanning_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of repos with code scanning (SAST) enabled"
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
  "org_security": {
    "two_factor_required": true
  },
  "repo_coverage_pct": 95.0,
  "branch_protection": {
    "pr_required_pct": 90.0,
    "approving_reviews_pct": 85.0,
    "status_checks_pct": 80.0,
    "signed_commits_pct": 10.0
  },
  "security_features": {
    "vuln_alerts_pct": 100,
    "secret_scanning_pct": 95.0,
    "code_scanning_pct": 60.0
  }
}
```

## 4. Field Mapping by Provider

| Normalized Field | GitHub | GitLab | Bitbucket |
|-----------------|--------|--------|-----------|
| `org_security.two_factor_required` | Org setting: require 2FA | Group setting: require 2FA | Workspace setting: require 2FA |
| `branch_protection.pr_required_pct` | Branch rule: require PR | Protected branch: require MR | Branch permission: require PR |
| `branch_protection.approving_reviews_pct` | Require approving reviews | Require approval | Require approvals |
| `branch_protection.status_checks_pct` | Require status checks | Require pipelines to succeed | Require builds to pass |
| `security_features.vuln_alerts_pct` | Dependabot alerts enabled | Dependency scanning enabled | — |
| `security_features.secret_scanning_pct` | Secret scanning enabled | Secret detection enabled | — |
| `security_features.code_scanning_pct` | Code scanning/CodeQL | SAST enabled | — |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Change management | `branch_protection.pr_required_pct` | `>= 90` |
| Code review required | `branch_protection.approving_reviews_pct` | `>= 90` |
| CI/CD enforcement | `branch_protection.status_checks_pct` | `>= 80` |
| Two-factor authentication | `org_security.two_factor_required` | `== true` |
| Vulnerability scanning | `security_features.vuln_alerts_pct` | `>= 90` |
| Secret detection | `security_features.secret_scanning_pct` | `>= 90` |
| Static analysis | `security_features.code_scanning_pct` | `>= 50` |

## 6. Repository Coverage

The `repo_coverage_pct` field indicates what percentage of the organization's repositories were included in the scan. This is important for understanding the completeness of the posture data.

Collectors SHOULD document their include/exclude patterns and explain why certain repositories may be excluded (e.g., archived repos, forks, specific patterns).
