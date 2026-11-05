# Access Posture Schema

**Type Identifier:** `evidencepack/access-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for access posture data from one system. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (GitHub, AWS, ClickHouse, etc.).

The artifact covers sign-in rules, members, admins, root accounts, machine credentials, and connected apps. A producer writes one artifact for each system it reads. `system_type` is the kind of system, such as `code_host` or `saas` (Section 4).

Only `schema_version`, `collected_at`, `provider`, and `system_type` are required. Every section and metric is optional. A value the collector could not read is absent or `null`.

A member is a person with an account. Service users and service accounts are counted under `credentials`. A member or admin is local when they can sign in without SSO, even if their account is linked to SSO. Local and SSO members can overlap when the system does not require SSO. The MFA percentages count only local members and admins. MFA for SSO sign-ins is set at the identity provider and is reported in [idp-posture.md](idp-posture.md).

A credential is unrestricted when it has neither an IP limit nor a scope narrower than full access. A credential with either limit is restricted.

Each percentage has its counts beside it, listed in the table below. Percentages are whole numbers. A percentage where more is better, such as `members.sso_coverage_pct`, rounds down. One where more is worse, such as `credentials.never_expiring_pct`, rounds up, so 1 credential of 201 that never expires reads 1. When a percentage's present flag is false, the percentage is `null`.

| Percentage | Numerator | Denominator | Present flag | Rounding |
|------------|-----------|-------------|--------------|----------|
| `members.sso_coverage_pct` | `members.sso_member_count` | `members.member_count` | `members.members_present` | Down |
| `members.local_mfa_coverage_pct` | `members.local_mfa_member_count` | `members.local_member_count` | `members.local_members_present` | Down |
| `members.inactive_pct` | `members.inactive_member_count` | `members.member_count` | `members.members_present` | Up |
| `admins.admin_share_pct` | `admins.admin_count` | `members.member_count` | `members.members_present` | Up |
| `admins.local_admin_mfa_coverage_pct` | `admins.local_mfa_admin_count` | `admins.local_admin_count` | `admins.local_admins_present` | Down |
| `root.root_mfa_coverage_pct` | `root.root_mfa_account_count` | `root.root_account_count` | `root.root_accounts_present` | Down |
| `credentials.never_expiring_pct` | `credentials.never_expiring_credential_count` | `credentials.credential_count` | `credentials.credentials_present` | Up |
| `credentials.unused_pct` | `credentials.unused_credential_count` | `credentials.credential_count` | `credentials.credentials_present` | Up |
| `credentials.unrestricted_pct` | `credentials.unrestricted_credential_count` | `credentials.credential_count` | `credentials.credentials_present` | Up |
| `credentials.write_access_pct` | `credentials.write_access_credential_count` | `credentials.credential_count` | `credentials.credentials_present` | Up |
| `credentials.rotation_overdue_pct` | `credentials.rotation_overdue_credential_count` | `credentials.credential_count` | `credentials.credentials_present` | Up |

An artifact is collected at the trust, audit, or internal level. `scopes` and the fields whose names end in `_count` are written at the audit and internal levels only. Every other field is written at every level, including the `_present` booleans and the percentages. The level is not a field of the artifact. The example in Section 3 is an audit-level artifact.

When a system has several accounts or subscriptions, each section holds totals across all of them. `scopes` lists each account or subscription separately (Section 6).

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "description": "Normalized access evidence for one system, such as GitHub, AWS, or ClickHouse. It covers sign-in rules, members, admins, root accounts, machine credentials, and connected apps. MFA here covers only people who sign in without SSO. The identity provider's MFA for SSO sign-ins is in idp-posture. When a system has several accounts or subscriptions, the sections hold totals across all of them, and scopes lists each one at the audit level. A *_present flag is then true when it is true in any scope. Each sign_in value is then false when it is false in any scope, null when it is null in any other, and true otherwise. Every metric is optional. Below the audit level, counts are left out, and the *_present flags are written instead",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider", "system_type"],
  "properties": {
    "schema_version": {
      "type": "string",
      "pattern": "^1\\.\\d+\\.\\d+$"
    },
    "collected_at": {
      "$ref": "#/$defs/timestamp"
    },
    "provider": {
      "type": "string",
      "minLength": 1,
      "description": "The system's vendor, such as github, aws, or clickhouse"
    },
    "system_type": {
      "enum": ["cloud", "code_host", "database", "platform", "monitoring", "saas"],
      "description": "The kind of system"
    },
    "sign_in": {
      "$ref": "#/$defs/sign_in",
      "description": "How people sign in. When the system has scopes, each value is false when it is false in any scope, null when it is null in any other, and true otherwise"
    },
    "members": {
      "$ref": "#/$defs/members"
    },
    "admins": {
      "$ref": "#/$defs/admins"
    },
    "root": {
      "$ref": "#/$defs/root"
    },
    "credentials": {
      "$ref": "#/$defs/credentials"
    },
    "connected_apps": {
      "$ref": "#/$defs/connected_apps"
    },
    "scopes": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/scope"
      },
      "description": "One entry per account or subscription. Each entry has the same sections, for that account or subscription alone"
    }
  },
  "additionalProperties": false,
  "$defs": {
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "count": {
      "type": ["integer", "null"],
      "minimum": 0
    },
    "pct": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100
    },
    "sign_in": {
      "type": ["object", "null"],
      "description": "How people sign in",
      "properties": {
        "sso_configured": {
          "type": ["boolean", "null"],
          "description": "Whether SSO is set up for the system"
        },
        "sso_required": {
          "type": ["boolean", "null"],
          "description": "Whether everyone must sign in through SSO"
        },
        "mfa_required": {
          "type": ["boolean", "null"],
          "description": "Whether the system requires MFA from anyone who signs in without SSO"
        },
        "scim_enabled": {
          "type": ["boolean", "null"],
          "description": "Whether accounts are created and removed through SCIM"
        }
      },
      "additionalProperties": false
    },
    "members": {
      "type": ["object", "null"],
      "description": "People with an account. Service users and service accounts count as credentials",
      "properties": {
        "members_present": {
          "type": ["boolean", "null"],
          "description": "Whether the system has any members. When false, the member percentages have no members to count"
        },
        "member_count": {
          "$ref": "#/$defs/count",
          "description": "Members"
        },
        "external_members_present": {
          "type": ["boolean", "null"],
          "description": "Whether any members come from outside the organization, such as outside collaborators or guests"
        },
        "external_member_count": {
          "$ref": "#/$defs/count",
          "description": "Members from outside the organization"
        },
        "sso_member_count": {
          "$ref": "#/$defs/count",
          "description": "Members whose account is linked to SSO"
        },
        "sso_coverage_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of members whose account is linked to SSO. Null when there are no members"
        },
        "local_members_present": {
          "type": ["boolean", "null"],
          "description": "Whether any members can sign in without SSO, with a password or a social login. A member linked to SSO counts too when the system still lets them sign in another way, so local and SSO members can overlap. When false, local_mfa_coverage_pct has no members to count"
        },
        "local_member_count": {
          "$ref": "#/$defs/count",
          "description": "Members who can sign in without SSO. A member linked to SSO counts when the system still lets them sign in another way"
        },
        "local_mfa_member_count": {
          "$ref": "#/$defs/count",
          "description": "Members counted in local_member_count who have MFA"
        },
        "local_mfa_coverage_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of members who can sign in without SSO and have MFA. Null when no member can sign in without SSO"
        },
        "inactive_member_count": {
          "$ref": "#/$defs/count",
          "description": "Members with no sign-in or other recorded activity in the last 90 days"
        },
        "inactive_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of members with no sign-in or other recorded activity in the last 90 days. Null when there are no members"
        }
      },
      "additionalProperties": false
    },
    "admins": {
      "type": ["object", "null"],
      "description": "Members who can administer the system",
      "properties": {
        "admins_present": {
          "type": ["boolean", "null"],
          "description": "Whether the system has any admins"
        },
        "admin_count": {
          "$ref": "#/$defs/count",
          "description": "Admins"
        },
        "admin_share_pct": {
          "$ref": "#/$defs/pct",
          "description": "Admins as a percentage of members. Null when there are no members"
        },
        "local_admins_present": {
          "type": ["boolean", "null"],
          "description": "Whether any admins can sign in without SSO. An admin linked to SSO counts too when the system still lets them sign in another way. When false, local_admin_mfa_coverage_pct has no admins to count"
        },
        "local_admin_count": {
          "$ref": "#/$defs/count",
          "description": "Admins who can sign in without SSO. An admin linked to SSO counts when the system still lets them sign in another way"
        },
        "local_mfa_admin_count": {
          "$ref": "#/$defs/count",
          "description": "Admins counted in local_admin_count who have MFA"
        },
        "local_admin_mfa_coverage_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of admins who can sign in without SSO and have MFA. Null when no admin can sign in without SSO"
        }
      },
      "additionalProperties": false
    },
    "root": {
      "type": ["object", "null"],
      "description": "Built-in accounts that bypass normal administration, such as the AWS root user",
      "properties": {
        "root_accounts_present": {
          "type": ["boolean", "null"],
          "description": "Whether the system has a built-in root account. When false, root_mfa_coverage_pct has no accounts to count"
        },
        "root_account_count": {
          "$ref": "#/$defs/count",
          "description": "Root accounts. A system with scopes has one per scope"
        },
        "root_mfa_account_count": {
          "$ref": "#/$defs/count",
          "description": "Root accounts with MFA"
        },
        "root_mfa_coverage_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of root accounts with MFA. Null when there are none"
        },
        "root_access_keys_present": {
          "type": ["boolean", "null"],
          "description": "Whether any root account has access keys"
        }
      },
      "additionalProperties": false
    },
    "credentials": {
      "type": ["object", "null"],
      "description": "API keys, tokens, deploy keys, service users, and access keys",
      "properties": {
        "credentials_present": {
          "type": ["boolean", "null"],
          "description": "Whether the system has any machine credentials. When false, the credential percentages have none to count"
        },
        "credential_count": {
          "$ref": "#/$defs/count",
          "description": "Machine credentials"
        },
        "never_expiring_credential_count": {
          "$ref": "#/$defs/count",
          "description": "Credentials that never expire"
        },
        "never_expiring_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of credentials that never expire. Null when there are none"
        },
        "unused_credential_count": {
          "$ref": "#/$defs/count",
          "description": "Credentials never used, or unused in the last 90 days"
        },
        "unused_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of credentials never used, or unused in the last 90 days. Null when there are none"
        },
        "unrestricted_credential_count": {
          "$ref": "#/$defs/count",
          "description": "Credentials with neither an IP limit nor a scope narrower than full access. A credential limited to chosen permissions or resources has a narrower scope. A credential with either limit is restricted"
        },
        "unrestricted_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of credentials with neither an IP limit nor a scope narrower than full access. Null when there are none"
        },
        "write_access_credential_count": {
          "$ref": "#/$defs/count",
          "description": "Credentials that can write or administer"
        },
        "write_access_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of credentials that can write or administer. Null when there are none"
        },
        "rotation_overdue_credential_count": {
          "$ref": "#/$defs/count",
          "description": "Credentials older than 90 days. When the system does not record when a credential was created, this is left out with a gap"
        },
        "rotation_overdue_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of credentials older than 90 days. Left out with a gap when the system does not record when a credential was created. Null when there are none"
        }
      },
      "additionalProperties": false
    },
    "connected_apps": {
      "type": ["object", "null"],
      "description": "Third-party apps with access to the system",
      "properties": {
        "apps_present": {
          "type": ["boolean", "null"],
          "description": "Whether any third-party app has access"
        },
        "app_count": {
          "$ref": "#/$defs/count",
          "description": "Third-party apps with access"
        }
      },
      "additionalProperties": false
    },
    "scope": {
      "type": "object",
      "required": ["id", "kind"],
      "properties": {
        "id": {
          "type": "string",
          "minLength": 1,
          "description": "The account or subscription ID"
        },
        "name": {
          "type": "string",
          "description": "The account or subscription name"
        },
        "kind": {
          "enum": ["account", "subscription"]
        },
        "sign_in": {
          "$ref": "#/$defs/sign_in"
        },
        "members": {
          "$ref": "#/$defs/members"
        },
        "admins": {
          "$ref": "#/$defs/admins"
        },
        "root": {
          "$ref": "#/$defs/root"
        },
        "credentials": {
          "$ref": "#/$defs/credentials"
        }
      },
      "additionalProperties": false
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
  "system_type": "code_host",
  "sign_in": {
    "sso_configured": true,
    "sso_required": false,
    "mfa_required": true,
    "scim_enabled": false
  },
  "members": {
    "members_present": true,
    "member_count": 60,
    "external_members_present": true,
    "external_member_count": 4,
    "sso_member_count": 53,
    "sso_coverage_pct": 88,
    "local_members_present": true,
    "local_member_count": 7,
    "local_mfa_member_count": 7,
    "local_mfa_coverage_pct": 100,
    "inactive_member_count": 5,
    "inactive_pct": 9
  },
  "admins": {
    "admins_present": true,
    "admin_count": 4,
    "admin_share_pct": 7,
    "local_admins_present": false,
    "local_admin_count": 0,
    "local_mfa_admin_count": 0,
    "local_admin_mfa_coverage_pct": null
  },
  "root": {
    "root_accounts_present": false,
    "root_account_count": 0,
    "root_mfa_account_count": 0,
    "root_mfa_coverage_pct": null,
    "root_access_keys_present": false
  },
  "credentials": {
    "credentials_present": true,
    "credential_count": 25,
    "never_expiring_credential_count": 8,
    "never_expiring_pct": 32,
    "unused_credential_count": 5,
    "unused_pct": 20,
    "unrestricted_credential_count": 10,
    "unrestricted_pct": 40,
    "write_access_credential_count": 12,
    "write_access_pct": 48,
    "rotation_overdue_credential_count": 9,
    "rotation_overdue_pct": 36
  },
  "connected_apps": {
    "apps_present": true,
    "app_count": 14
  }
}
```

## 4. System Types

| `system_type` | Kind of system |
|---------------|----------------|
| `cloud` | A cloud provider, such as AWS |
| `code_host` | A source code host, such as GitHub |
| `database` | A managed database service, such as ClickHouse |
| `platform` | A platform that hosts or runs the organization's software |
| `monitoring` | A monitoring, logging, or error tracking service |
| `saas` | Any other software service |

`root` covers built-in accounts that bypass normal administration, such as the AWS root user. On a system with no such account, `root.root_accounts_present` is false. `credentials` covers API keys, tokens, deploy keys, service users, and access keys.

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| SSO enforcement | `sign_in.sso_required` | `== true` |
| MFA for local sign-in | `sign_in.mfa_required`, `members.local_mfa_coverage_pct` | `== true`, `== 100` |
| MFA for admins | `admins.local_admin_mfa_coverage_pct` | `== 100` |
| Root account protection | `root.root_mfa_coverage_pct`, `root.root_access_keys_present` | `== 100`, `== false` |
| Automated provisioning | `sign_in.scim_enabled` | `== true` |
| Inactive accounts | `members.inactive_pct` | `<= 5` |
| Privileged access minimization | `admins.admin_share_pct` | `<= 10` |
| Credential expiry | `credentials.never_expiring_pct` | `== 0` |
| Credential rotation | `credentials.rotation_overdue_pct` | `<= 10` |
| Unused credentials | `credentials.unused_pct` | `== 0` |

A pack holds one access-posture artifact for each system. A clause that should hold on every system sets `artifacts: all` ([profiles.md](../profiles.md) Section 4.4).

## 6. Multiple Accounts

When a system has several accounts or subscriptions, such as the accounts in an AWS organization, each top-level section holds totals across all of them. `scopes` has one entry for each account or subscription. Each entry has an `id`, a `kind` of `account` or `subscription`, and an optional `name`. It has the `sign_in`, `members`, `admins`, `root`, and `credentials` sections for that account alone.

A top-level `sign_in` value is false when it is false in any scope. Otherwise it is `null` when it is `null` in any scope, and true when it is true in every scope. A `_present` flag is true when it is true in any scope. `root.root_account_count` counts one root account for each scope. `scopes` is written at the audit and internal levels only, so a trust-level artifact has the totals alone.
