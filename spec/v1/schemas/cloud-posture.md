# Cloud Infrastructure Posture Schema

**Type Identifier:** `evidencepack/cloud-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for cloud infrastructure security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (AWS, GCP, Azure, etc.).

The sign-in rules, members, root accounts, and machine credentials of these accounts are in [access-posture.md](access-posture.md). Scanner coverage and open findings are in [vuln-scanning.md](vuln-scanning.md).

Only `schema_version`, `collected_at`, `provider`, and `accounts` are required. A value the collector could not read is absent or `null`. A percentage over an empty population is `null`. Where the schema lists the count behind a percentage, such as `storage.storage_resource_count` for `storage.encryption_pct`, that count is 0.

Each percentage has its count beside it. `storage.encryption_pct` is `encrypted_storage_resource_count` out of `storage_resource_count`, and the database and network percentages follow the same pattern. Percentages are whole numbers. A percentage where more is better, such as `storage.encryption_pct`, rounds down. One where more is worse, such as `network.ssh_open_to_world_pct`, rounds up, so 1 security group of 201 open to the world reads 1. The counts are optional. When they are absent, the `_present` booleans show whether a section had anything to count. When one is false, that section's percentages are `null`.

An artifact is collected at the trust, audit, or internal level. `accounts_expected`, each account's `account_id`, `database.multi_az_pct`, and the fields whose names end in `_count` are written at the audit and internal levels only. Every other field is written at every level, including the `_present` booleans and the other percentages. The level is not a field of the artifact. The example in Section 3 is an audit-level artifact.

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
    "accounts_expected": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Number of accounts the collection was configured to cover"
    },
    "all_accounts_collected": {
      "type": ["boolean", "null"],
      "description": "Whether every account the collection was configured to cover answered"
    },
    "accounts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "account_id": {
            "type": "string",
            "description": "Account/project/subscription identifier"
          },
          "storage": {
            "type": ["object", "null"],
            "properties": {
              "storage_resources_present": {
                "type": ["boolean", "null"],
                "description": "Whether the account has any storage resources. When false, the storage percentages are null"
              },
              "storage_resource_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Storage resources the storage percentages count, such as S3 buckets or Azure storage accounts"
              },
              "encrypted_storage_resource_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Storage resources with encryption at rest"
              },
              "public_access_blocked_storage_resource_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Storage resources with public access blocked"
              },
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
          "database": {
            "type": ["object", "null"],
            "properties": {
              "databases_present": {
                "type": ["boolean", "null"],
                "description": "Whether the account has any managed databases, such as RDS instances, Aurora clusters, or Azure SQL databases. When false, the database percentages are null"
              },
              "database_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Managed databases in the account. The database percentages count these"
              },
              "encrypted_database_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Databases encrypted at rest, judged by the storage encryption type where the provider reports one"
              },
              "encryption_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of databases encrypted at rest. Null when the account has no databases"
              },
              "public_access_blocked_database_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Databases that cannot be reached from the public internet"
              },
              "public_access_blocked_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of databases that cannot be reached from the public internet. Null when the account has no databases"
              },
              "multi_az_database_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Databases that run across availability zones. For Aurora, this is read from the cluster"
              },
              "multi_az_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of databases that run across availability zones. It is evidence about configuration only and does not show that recovery works. Null when the account has no databases"
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
              "security_groups_present": {
                "type": ["boolean", "null"],
                "description": "Whether the account has any security groups or network security groups, or firewall rules on Google Cloud. When false, the network percentages are null"
              },
              "security_group_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Security groups or network security groups, or firewall rules on Google Cloud. The network percentages count these"
              },
              "ssh_open_security_group_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Security groups with a rule that allows SSH (port 22) from any address"
              },
              "rdp_open_security_group_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Security groups with a rule that allows RDP (port 3389) from any address"
              },
              "ssh_open_to_world_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of security groups or network security groups (firewall rules on Google Cloud) with a rule that allows SSH (port 22) from any address"
              },
              "rdp_open_to_world_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "description": "Percentage of security groups or network security groups (firewall rules on Google Cloud) with a rule that allows RDP (port 3389) from any address"
              }
            }
          },
          "backup": {
            "type": ["object", "null"],
            "properties": {
              "retention_days_min": {
                "type": ["integer", "null"],
                "minimum": 0,
                "description": "Minimum backup retention period across all backup-enabled resources (days). Null when no resource has backups enabled. Left out with a gap when a read failed"
              }
            }
          },
          "threat_detection": {
            "type": ["object", "null"],
            "properties": {
              "enabled": {
                "type": ["boolean", "null"],
                "description": "True when threat detection, such as Amazon GuardDuty, is on in every enabled region"
              }
            }
          },
          "iam": {
            "type": ["object", "null"],
            "deprecated": true,
            "description": "Kept for profiles written before epack v0.10.0, and to be removed in a later release. These values are in evidencepack/access-posture@v1 now",
            "properties": {
              "user_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "deprecated": true,
                "description": "Number of human user identities that can sign in to the account. mfa_coverage_pct counts these users. The same value is members.local_member_count in evidencepack/access-posture@v1"
              },
              "mfa_coverage_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "deprecated": true,
                "description": "Percentage of human user identities that can sign in to the account and have MFA enabled. The same value is members.local_mfa_coverage_pct in evidencepack/access-posture@v1"
              },
              "root_mfa_enabled": {
                "type": ["boolean", "null"],
                "deprecated": true,
                "description": "Whether the account's highest-privilege owner identity (the AWS root user, a Google Cloud organization administrator, an Azure Global Administrator) has MFA enabled. In evidencepack/access-posture@v1, it is true when root.root_mfa_coverage_pct is 100"
              },
              "root_access_protected": {
                "type": ["boolean", "null"],
                "deprecated": true,
                "description": "Whether the account's highest-privilege owner identity (the AWS root user, a Google Cloud organization administrator, an Azure Global Administrator) is protected from password-only use, by MFA or by having no standing credentials. In evidencepack/access-posture@v1, it is true when root.root_mfa_coverage_pct is 100 and root.root_access_keys_present is false"
              },
              "access_key_user_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "deprecated": true,
                "description": "Number of users with an active long-lived programmatic credential (access keys, service account keys, client secrets). access_key_rotation_compliant_pct counts these users. The same value is credentials.credential_count in evidencepack/access-posture@v1"
              },
              "access_key_rotation_compliant_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "deprecated": true,
                "description": "Percentage of users with an active long-lived programmatic credential (access keys, service account keys, client secrets) whose active credentials are all rotated within policy (typically 90 days). In evidencepack/access-posture@v1, credentials.rotation_overdue_pct counts the keys not rotated in time instead"
              },
              "saml_provider_count": {
                "type": ["integer", "null"],
                "minimum": 0,
                "deprecated": true,
                "description": "Number of SAML identity providers the account trusts directly for sign-in. A provider created by a workforce sign-in service such as AWS IAM Identity Center is not counted. Nothing in evidencepack/access-posture@v1 replaces it"
              }
            }
          },
          "vuln_scanning": {
            "type": ["object", "null"],
            "deprecated": true,
            "description": "Kept for profiles written before epack v0.10.0, and to be removed in a later release. These values are in evidencepack/vuln-scanning@v1 now",
            "properties": {
              "enabled": {
                "type": ["boolean", "null"],
                "deprecated": true,
                "description": "Whether vulnerability scanning is enabled (Inspector/Security Command Center/Defender). In evidencepack/vuln-scanning@v1, each entry in scanners has its own enabled value"
              },
              "unpatched_pct": {
                "type": ["number", "null"],
                "minimum": 0,
                "maximum": 100,
                "deprecated": true,
                "description": "Percentage of scanned resources with unpatched vulnerabilities. Nothing in evidencepack/vuln-scanning@v1 replaces it"
              }
            }
          }
        }
      }
    },
    "workforce_sign_in": {
      "type": ["object", "null"],
      "deprecated": true,
      "description": "A central sign-in service that signs people in to the accounts, such as AWS IAM Identity Center. Left out when the collector does not report one. Kept for profiles written before epack v0.10.0, and to be removed in a later release. These values are in evidencepack/access-posture@v1 now",
      "properties": {
        "in_use": {
          "type": ["boolean", "null"],
          "deprecated": true,
          "description": "Whether people can sign in to any of the accounts through the workforce sign-in service. The same value is sign_in.sso_configured in evidencepack/access-posture@v1"
        },
        "user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "deprecated": true,
          "description": "Number of users in the workforce sign-in service's directory. The same value is members.sso_member_count in evidencepack/access-posture@v1"
        },
        "mfa_source": {
          "type": ["string", "null"],
          "enum": ["sign_in_service", "external_idp", "unknown", null],
          "deprecated": true,
          "description": "Where the workforce sign-in service's users have MFA set. It is sign_in_service when the service itself sets it, external_idp when an external identity provider that the users come from sets it, and unknown when the collector cannot tell. Nothing in evidencepack/access-posture@v1 replaces it"
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
  "accounts_expected": 1,
  "all_accounts_collected": true,
  "accounts": [
    {
      "account_id": "123456789012",
      "storage": {
        "storage_resources_present": true,
        "storage_resource_count": 42,
        "encrypted_storage_resource_count": 42,
        "public_access_blocked_storage_resource_count": 42,
        "encryption_pct": 100,
        "public_access_blocked_pct": 100
      },
      "database": {
        "databases_present": true,
        "database_count": 6,
        "encrypted_database_count": 6,
        "encryption_pct": 100,
        "public_access_blocked_database_count": 6,
        "public_access_blocked_pct": 100,
        "multi_az_database_count": 4,
        "multi_az_pct": 66
      },
      "logging": {
        "audit_log_enabled": true,
        "audit_log_all_regions": true,
        "cloudtrail_enabled": true,
        "cloudtrail_multiregion": true
      },
      "network": {
        "security_groups_present": true,
        "security_group_count": 37,
        "ssh_open_security_group_count": 0,
        "rdp_open_security_group_count": 0,
        "ssh_open_to_world_pct": 0,
        "rdp_open_to_world_pct": 0
      },
      "backup": {
        "retention_days_min": 7
      },
      "threat_detection": {
        "enabled": true
      }
    }
  ]
}
```

## 4. Field Mapping by Provider

| Normalized Field | AWS | Azure |
|-----------------|-----|-------|
| `storage.encryption_pct` | S3 bucket encryption | Storage accounts encrypted with customer-managed keys |
| `storage.public_access_blocked_pct` | S3 Block Public Access | Storage accounts with public access disabled |
| `logging.audit_log_enabled` | CloudTrail trail logging | Activity Log exported |
| `logging.audit_log_all_regions` | Multi-region trail logging | Same as `audit_log_enabled` (the log is global) |
| `network.ssh_open_to_world_pct` | Security group 0.0.0.0/0:22 | NSG 0.0.0.0/0:22 |
| `network.rdp_open_to_world_pct` | Security group 0.0.0.0/0:3389 | NSG 0.0.0.0/0:3389 |
| `backup.retention_days_min` | Shortest RDS backup retention | Shortest retention in Recovery Services vault backup policies |

Each count beside a percentage comes from the same source as the percentage. `database` covers managed databases, such as RDS instances, Aurora clusters, and Azure SQL databases. `threat_detection.enabled` is true when a service such as Amazon GuardDuty is on in every enabled region.

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Audit logging enabled | `logging.audit_log_enabled` | `== true` |
| Audit logging in every region | `logging.audit_log_all_regions` | `== true` |
| Default-deny firewall | `network.ssh_open_to_world_pct`, `network.rdp_open_to_world_pct` | `== 0` |
| Backup retention | `backup.retention_days_min` | `>= 7` |
| Storage encryption | `storage.encryption_pct` | `>= 100` |
| Database encryption | `database.encryption_pct` | `>= 100` |
| Threat detection | `threat_detection.enabled` | `== true` |

Root account protection, MFA, and credential rotation for the accounts are mapped in [access-posture.md](access-posture.md) Section 5. Vulnerability scanning is mapped in [vuln-scanning.md](vuln-scanning.md) Section 5.

## 6. Multi-Account Considerations

When an organization has multiple cloud accounts, the `accounts` array contains one entry per account. Profile conditions can reference specific accounts by index (`accounts[0]`) or aggregate across all accounts.

`accounts_expected` is the number of accounts the collection was configured to cover. `all_accounts_collected` is true when the collection read every one of them. When it is false, the `accounts` array is missing some accounts, and an aggregate over it covers only the accounts that were read.

A condition over every account uses a path such as `$.accounts[*].storage.encryption_pct` and sets `cardinality` ([profiles.md](../profiles.md) Section 4.5). With `all`, every account must satisfy the condition. With `any`, one account is enough. With `none`, no account may satisfy it.

## 7. Versioning and Deprecated Fields

`schema_version` is `1.0.0` in every v1 artifact. Changes to this schema within v1 add optional fields. While the schema is a draft, a change within v1 can also move a field to another type. A consumer ignores the fields it does not know. It would reject any other `schema_version`, so changing the value would break it. A consumer that needs a newer field detects it by its presence.

A deprecated field keeps its meaning and type and stays valid for all of v1. The v2 schema removes it. Producers SHOULD emit each deprecated field alongside its replacement until v2, so profiles and consumers written against the deprecated name keep working, and when both are present they MUST hold the same value. Consumers SHOULD read the replacement and MAY fall back to the deprecated field when the replacement is absent, as it is in artifacts from producers that predate it.

| Deprecated field | Replacement |
|-----------------|-------------|
| `logging.cloudtrail_enabled` | `logging.audit_log_enabled` |
| `logging.cloudtrail_multiregion` | `logging.audit_log_all_regions` |

epack v0.10.0 moved `accounts[].iam`, `accounts[].vuln_scanning`, and `workforce_sign_in` to [access-posture.md](access-posture.md) and [vuln-scanning.md](vuln-scanning.md). They are deprecated here too, and producers write them while profiles move to the new fields. Unlike the fields above, they are removed in a later revision of the v1 draft. Each replacement sits in another artifact. An account's own value is in the entry for that account in the artifact's `scopes`.

A producer that leaves the replacement's artifact out of a pack MUST leave out the deprecated fields that moved to it. A pack with no access-posture artifact then has no `iam` or `workforce_sign_in`, and one with no vuln-scanning artifact has no `vuln_scanning`. A pack never carries a value its publisher chose not to share.

| Deprecated field | Replacement |
|-----------------|-------------|
| `accounts[].iam.user_count` | access-posture `members.local_member_count` |
| `accounts[].iam.mfa_coverage_pct` | access-posture `members.local_mfa_coverage_pct` |
| `accounts[].iam.root_mfa_enabled` | access-posture `root.root_mfa_coverage_pct` at 100 |
| `accounts[].iam.root_access_protected` | access-posture `root.root_mfa_coverage_pct` at 100 with `root.root_access_keys_present` false |
| `accounts[].iam.access_key_user_count` | access-posture `credentials.credential_count` |
| `accounts[].iam.access_key_rotation_compliant_pct` | access-posture `credentials.rotation_overdue_pct`, which counts the keys not rotated in time |
| `accounts[].iam.saml_provider_count` | None |
| `accounts[].vuln_scanning.enabled` | vuln-scanning `scanners[].enabled` |
| `accounts[].vuln_scanning.unpatched_pct` | None |
| `workforce_sign_in.in_use` | access-posture `sign_in.sso_configured` |
| `workforce_sign_in.user_count` | access-posture `members.sso_member_count` |
| `workforce_sign_in.mfa_source` | None |
