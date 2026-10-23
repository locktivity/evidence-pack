# Cloud Infrastructure Posture Schema

**Type Identifier:** `evidencepack/cloud-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for cloud infrastructure security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (AWS, GCP, Azure, etc.).

Only `schema_version`, `collected_at`, `provider`, `accounts`, and each account's `account_id` are required. A value the collector could not read is absent or `null`. A percentage over an empty population is absent. Where the schema lists the count behind a percentage, such as `iam.user_count` for `iam.mfa_coverage_pct`, that count is 0.

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
            "type": ["object", "null"],
            "properties": {
              "user_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Number of human user identities that can sign in to the account. mfa_coverage_pct counts these users"
              },
              "mfa_coverage_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of human user identities that can sign in to the account and have MFA enabled"
              },
              "root_mfa_enabled": {
                "type": ["boolean", "null"],
                "description": "Whether the account's highest-privilege owner identity (the AWS root user, a Google Cloud organization administrator, an Azure Global Administrator) has MFA enabled"
              },
              "root_access_protected": {
                "type": ["boolean", "null"],
                "description": "Whether the account's highest-privilege owner identity (the AWS root user, a Google Cloud organization administrator, an Azure Global Administrator) is protected from password-only use, by MFA or by having no standing credentials"
              },
              "access_key_user_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Number of users with an active long-lived programmatic credential (access keys, service account keys, client secrets). access_key_rotation_compliant_pct counts these users"
              },
              "access_key_rotation_compliant_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of users with an active long-lived programmatic credential (access keys, service account keys, client secrets) whose active credentials are all rotated within policy (typically 90 days)"
              },
              "saml_provider_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Number of SAML identity providers the account trusts directly for sign-in. A provider created by a workforce sign-in service such as AWS IAM Identity Center is not counted"
              }
            }
          },
          "storage": {
            "type": ["object", "null"],
            "properties": {
              "encryption_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of storage resources with encryption at rest"
              },
              "public_access_blocked_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of storage resources with public access blocked"
              }
            }
          },
          "logging": {
            "type": ["object", "null"],
            "properties": {
              "audit_log_enabled": {
                "type": ["boolean", "null"],
                "description": "Whether an account-wide audit log of management activity is enabled, for example CloudTrail, Cloud Audit Logs, or an exported Azure Activity Log"
              },
              "audit_log_all_regions": {
                "type": ["boolean", "null"],
                "description": "Whether the audit log covers every region the account uses. True where the provider's audit log is global"
              },
              "cloudtrail_enabled": {
                "type": ["boolean", "null"],
                "deprecated": true,
                "description": "Deprecated alias of audit_log_enabled, with the same value. Removed in v2"
              },
              "cloudtrail_multiregion": {
                "type": ["boolean", "null"],
                "deprecated": true,
                "description": "Deprecated alias of audit_log_all_regions, with the same value. Removed in v2"
              }
            }
          },
          "network": {
            "type": ["object", "null"],
            "properties": {
              "ssh_open_to_world_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of network rules (security groups, network security groups, firewall rules) allowing SSH (port 22) from any address"
              },
              "rdp_open_to_world_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of network rules (security groups, network security groups, firewall rules) allowing RDP (port 3389) from any address"
              }
            }
          },
          "backup": {
            "type": ["object", "null"],
            "properties": {
              "retention_days_min": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Minimum backup retention period across all backup-enabled resources (days)"
              }
            }
          },
          "vuln_scanning": {
            "type": ["object", "null"],
            "properties": {
              "enabled": {
                "type": ["boolean", "null"],
                "description": "Whether vulnerability scanning is enabled (Inspector/Security Command Center/Defender)"
              },
              "unpatched_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of scanned resources with unpatched vulnerabilities"
              }
            }
          }
        }
      }
    },
    "workforce_sign_in": {
      "type": ["object", "null"],
      "description": "A central sign-in service that signs people in to the accounts, such as AWS IAM Identity Center. Left out when the collector does not report one",
      "properties": {
        "in_use": {
          "type": ["boolean", "null"],
          "description": "Whether people can sign in to any of the accounts through the workforce sign-in service"
        },
        "user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of users in the workforce sign-in service's directory"
        },
        "mfa_source": {
          "type": ["string", "null"],
          "enum": ["sign_in_service", "external_idp", "unknown", null],
          "description": "Where the workforce sign-in service's users have MFA set. It is sign_in_service when the service itself sets it, external_idp when an external identity provider that the users come from sets it, and unknown when the collector cannot tell"
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
        "user_count": 25,
        "mfa_coverage_pct": 100,
        "root_mfa_enabled": true,
        "root_access_protected": true,
        "access_key_user_count": 20,
        "access_key_rotation_compliant_pct": 95.0,
        "saml_provider_count": 0
      },
      "storage": {
        "encryption_pct": 100,
        "public_access_blocked_pct": 100
      },
      "logging": {
        "audit_log_enabled": true,
        "audit_log_all_regions": true,
        "cloudtrail_enabled": true,
        "cloudtrail_multiregion": true
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
  ],
  "workforce_sign_in": {
    "in_use": true,
    "user_count": 40,
    "mfa_source": "external_idp"
  }
}
```

## 4. Field Mapping by Provider

| Normalized Field | AWS | GCP | Azure |
|-----------------|-----|-----|-------|
| `iam.user_count` | IAM users | Cloud Identity users | Entra ID users |
| `iam.mfa_coverage_pct` | IAM users with MFA | Cloud Identity users with 2FA | Entra ID users with MFA |
| `iam.root_mfa_enabled` | Root account MFA | Organization admin MFA | Global admin MFA |
| `iam.root_access_protected` | Root MFA active or no standing root credentials | Organization admin protected from password-only use | Global admin protected from password-only use |
| `iam.access_key_user_count` | IAM users with an active access key | Service accounts with an active user-managed key | App registrations with an active client secret |
| `iam.saml_provider_count` | IAM SAML providers, not counting IAM Identity Center's | Not defined | Not defined |
| `logging.audit_log_enabled` | CloudTrail trail logging | Cloud Audit Logs enabled | Activity Log exported |
| `logging.audit_log_all_regions` | Multi-region trail logging | Same as `audit_log_enabled` (the log is global) | Same as `audit_log_enabled` (the log is global) |
| `network.ssh_open_to_world_pct` | Security group 0.0.0.0/0:22 | Firewall rule 0.0.0.0/0:22 | NSG 0.0.0.0/0:22 |
| `storage.encryption_pct` | S3 bucket encryption | GCS bucket encryption | Storage account encryption |
| `vuln_scanning.enabled` | Inspector enabled | Security Command Center | Defender for Cloud |
| `workforce_sign_in.in_use` | IAM Identity Center organization instance in use | Not defined | Not defined |
| `workforce_sign_in.user_count` | Users in the IAM Identity Center directory | Not defined | Not defined |
| `workforce_sign_in.mfa_source` | MFA set in IAM Identity Center or at an external identity provider | Not defined | Not defined |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Audit logging enabled | `logging.audit_log_enabled` | `== true` |
| Audit logging in every region | `logging.audit_log_all_regions` | `== true` |
| Default-deny firewall | `network.ssh_open_to_world_pct`, `network.rdp_open_to_world_pct` | `== 0` |
| Backup retention | `backup.retention_days_min` | `>= 7` |
| Vulnerability scanning | `vuln_scanning.enabled` | `== true` |
| Root access protection | `iam.root_access_protected` | `== true` |
| IAM MFA coverage | `iam.mfa_coverage_pct` | `>= 100` |
| Storage encryption | `storage.encryption_pct` | `>= 100` |

## 6. Multi-Account Considerations

When an organization has multiple cloud accounts, the `accounts` array contains one entry per account. Profile conditions can reference specific accounts by index (`accounts[0]`) or aggregate across all accounts.

For aggregate conditions, profiles SHOULD specify the aggregation method:
- `all`: Condition must pass for ALL accounts
- `any`: Condition must pass for AT LEAST ONE account
- `average`: Condition applies to the average across accounts

## 7. Versioning and Deprecated Fields

Changes to this schema within v1 only add optional fields, so `schema_version` is `1.0.0` in every v1 artifact. A consumer reads an artifact that has fields it does not know by ignoring them; it would reject any other `schema_version`, so changing the value would break it. A consumer that needs a newer field detects it by its presence.

A deprecated field keeps its meaning and type and stays valid for all of v1. The v2 schema removes it. Producers SHOULD emit each deprecated field alongside its replacement until v2, so profiles and consumers written against the deprecated name keep working, and when both are present they MUST hold the same value. Consumers SHOULD read the replacement and MAY fall back to the deprecated field when the replacement is absent, as it is in artifacts from producers that predate it.

| Deprecated field | Replacement |
|-----------------|-------------|
| `logging.cloudtrail_enabled` | `logging.audit_log_enabled` |
| `logging.cloudtrail_multiregion` | `logging.audit_log_all_regions` |
