# Cloud Infrastructure Posture Schema

**Type Identifier:** `evidencepack/cloud-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for cloud infrastructure security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (AWS, GCP, Azure, etc.).

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider", "accounts"],
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
      "enum": ["aws", "gcp", "azure", "other"],
      "description": "Cloud provider vendor"
    },
    "accounts": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["account_id"],
        "properties": {
          "account_id": {
            "type": "string",
            "description": "Account/project/subscription identifier"
          },
          "iam": {
            "type": "object",
            "properties": {
              "mfa_coverage_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of IAM users with MFA enabled"
              },
              "root_mfa_enabled": {
                "type": "boolean",
                "description": "Whether root/owner account has MFA enabled"
              },
              "access_key_rotation_compliant_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of access keys rotated within policy (typically 90 days)"
              }
            }
          },
          "storage": {
            "type": "object",
            "properties": {
              "encryption_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of storage resources with encryption at rest"
              },
              "public_access_blocked_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of storage resources with public access blocked"
              }
            }
          },
          "logging": {
            "type": "object",
            "properties": {
              "cloudtrail_enabled": {
                "type": "boolean",
                "description": "Whether audit logging is enabled (CloudTrail/Cloud Audit Logs/Activity Log)"
              },
              "cloudtrail_multiregion": {
                "type": "boolean",
                "description": "Whether audit logging covers all regions"
              },
              "flow_logs_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of VPCs/VNets with flow logging enabled"
              }
            }
          },
          "network": {
            "type": "object",
            "properties": {
              "ssh_open_to_world_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of security groups allowing SSH (port 22) from 0.0.0.0/0"
              },
              "rdp_open_to_world_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of security groups allowing RDP (port 3389) from 0.0.0.0/0"
              }
            }
          },
          "backup": {
            "type": "object",
            "properties": {
              "retention_days_min": {
                "type": "integer",
                "minimum": 0,
                "description": "Minimum backup retention period across all backup-enabled resources (days)"
              }
            }
          },
          "vuln_scanning": {
            "type": "object",
            "properties": {
              "enabled": {
                "type": "boolean",
                "description": "Whether vulnerability scanning is enabled (Inspector/Security Command Center/Defender)"
              },
              "unpatched_pct": {
                "type": "number",
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of scanned resources with unpatched vulnerabilities"
              }
            }
          }
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
  "provider": "aws",
  "accounts": [
    {
      "account_id": "123456789012",
      "iam": {
        "mfa_coverage_pct": 100,
        "root_mfa_enabled": true,
        "access_key_rotation_compliant_pct": 95.0
      },
      "storage": {
        "encryption_pct": 100,
        "public_access_blocked_pct": 100
      },
      "logging": {
        "cloudtrail_enabled": true,
        "cloudtrail_multiregion": true,
        "flow_logs_pct": 80.0
      },
      "network": {
        "ssh_open_to_world_pct": 0,
        "rdp_open_to_world_pct": 0
      },
      "backup": {
        "retention_days_min": 7
      },
      "vuln_scanning": {
        "enabled": true,
        "unpatched_pct": 5.0
      }
    }
  ]
}
```

## 4. Field Mapping by Provider

| Normalized Field | AWS | GCP | Azure |
|-----------------|-----|-----|-------|
| `iam.mfa_coverage_pct` | IAM users with MFA | Cloud Identity users with 2FA | Entra ID users with MFA |
| `iam.root_mfa_enabled` | Root account MFA | Organization admin MFA | Global admin MFA |
| `logging.cloudtrail_enabled` | CloudTrail enabled | Cloud Audit Logs enabled | Activity Log enabled |
| `logging.cloudtrail_multiregion` | Multi-region trail | Org-level logging | Subscription-level logging |
| `network.ssh_open_to_world_pct` | Security group 0.0.0.0/0:22 | Firewall rule 0.0.0.0/0:22 | NSG 0.0.0.0/0:22 |
| `storage.encryption_pct` | S3 bucket encryption | GCS bucket encryption | Storage account encryption |
| `vuln_scanning.enabled` | Inspector enabled | Security Command Center | Defender for Cloud |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Audit logging enabled | `logging.cloudtrail_enabled` | `== true` |
| Multi-region audit logging | `logging.cloudtrail_multiregion` | `== true` |
| Default-deny firewall | `network.ssh_open_to_world_pct`, `network.rdp_open_to_world_pct` | `== 0` |
| Backup retention | `backup.retention_days_min` | `>= 7` |
| Vulnerability scanning | `vuln_scanning.enabled` | `== true` |
| Root MFA | `iam.root_mfa_enabled` | `== true` |
| IAM MFA coverage | `iam.mfa_coverage_pct` | `>= 100` |
| Storage encryption | `storage.encryption_pct` | `>= 100` |

## 6. Multi-Account Considerations

When an organization has multiple cloud accounts, the `accounts` array contains one entry per account. Profile conditions can reference specific accounts by index (`accounts[0]`) or aggregate across all accounts.

For aggregate conditions, profiles SHOULD specify the aggregation method:
- `all`: Condition must pass for ALL accounts
- `any`: Condition must pass for AT LEAST ONE account
- `average`: Condition applies to the average across accounts
