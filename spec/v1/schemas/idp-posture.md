# Identity Provider Posture Schema

**Type Identifier:** `evidencepack/idp-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for identity provider security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (Okta, Google Workspace, Ping Identity, Microsoft Entra ID, etc.).

Core tenant identity signals are required. Richer posture sections are optional and SHOULD be emitted when the provider exposes them.

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider", "org_domain", "user_security"],
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
      "enum": ["okta", "ping", "entra", "google_workspace", "auth0", "onelogin", "other"],
      "description": "Identity provider vendor"
    },
    "org_domain": {
      "type": "string",
      "description": "Organization domain or tenant identifier"
    },
    "user_security": {
      "type": "object",
      "required": [
        "mfa_coverage_pct",
        "mfa_phishing_resistant_pct",
        "inactive_pct",
        "locked_out_pct"
      ],
      "properties": {
        "mfa_coverage_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users with any MFA factor enrolled"
        },
        "mfa_phishing_resistant_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users with phishing-resistant MFA (WebAuthn/FIDO2)"
        },
        "inactive_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users inactive for 90+ days"
        },
        "locked_out_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users currently locked out"
        },
        "weak_password_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users with weak passwords"
        },
        "password_policy_noncompliant_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users whose passwords do not meet current password policy requirements"
        }
      }
    },
    "app_security": {
      "type": "object",
      "properties": {
        "sso_coverage_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of applications using federated SSO (SAML/OIDC)"
        },
        "provisioning_enabled_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of applications with automated provisioning"
        },
        "deprovisioning_enabled_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of applications with automated deprovisioning"
        },
        "authorized_third_party_apps_count": {
          "type": "integer",
          "minimum": 0,
          "description": "Count of authorized third-party applications or OAuth integrations"
        }
      }
    },
    "privileged_access": {
      "type": "object",
      "properties": {
        "privileged_users_count": {
          "type": "integer",
          "minimum": 0,
          "description": "Count of users with privileged administrative access"
        },
        "super_admin_count": {
          "type": "integer",
          "minimum": 0,
          "description": "Count of highest-privilege administrative accounts (for example Global Admin or Super Admin)"
        },
        "privileged_mfa_coverage_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of privileged users with MFA enabled or enforced"
        },
        "privileged_phishing_resistant_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of privileged users with phishing-resistant MFA"
        },
        "standing_privileged_users_count": {
          "type": "integer",
          "minimum": 0,
          "description": "Count of users with standing privileged access instead of just-in-time or eligible access"
        }
      }
    },
    "lifecycle": {
      "type": "object",
      "properties": {
        "suspended_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of accounts in suspended or disabled state"
        },
        "archived_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of accounts in archived, retained, or soft-deleted state"
        }
      }
    },
    "policy": {
      "type": "object",
      "properties": {
        "mfa_required": {
          "type": "boolean",
          "description": "Whether the tenant enforces MFA for all covered users"
        },
        "mfa_required_coverage_pct": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users covered by policies that require MFA"
        },
        "session_lifetime_max_min": {
          "type": "integer",
          "minimum": 0,
          "description": "Maximum session lifetime across all policies (minutes)"
        },
        "idle_timeout_max_min": {
          "type": "integer",
          "minimum": 0,
          "description": "Maximum idle timeout across all policies (minutes)"
        },
        "legacy_auth_blocked": {
          "type": "boolean",
          "description": "Whether legacy authentication protocols are blocked"
        },
        "phishing_resistant_required_for_privileged": {
          "type": "boolean",
          "description": "Whether phishing-resistant authentication is required for privileged users"
        }
      }
    },
    "device_access": {
      "type": "object",
      "properties": {
        "managed_device_required": {
          "type": "boolean",
          "description": "Whether access to core corporate resources requires a managed or compliant device"
        },
        "managed_device_required_for_admins": {
          "type": "boolean",
          "description": "Whether privileged access requires a managed or compliant device"
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
  "provider": "okta",
  "org_domain": "acme.okta.com",
  "user_security": {
    "mfa_coverage_pct": 98.5,
    "mfa_phishing_resistant_pct": 45.0,
    "inactive_pct": 2.3,
    "locked_out_pct": 0.5,
    "weak_password_pct": 3.1,
    "password_policy_noncompliant_pct": 1.2
  },
  "app_security": {
    "sso_coverage_pct": 85.0,
    "provisioning_enabled_pct": 60.0,
    "deprovisioning_enabled_pct": 58.0,
    "authorized_third_party_apps_count": 41
  },
  "privileged_access": {
    "privileged_users_count": 8,
    "super_admin_count": 3,
    "privileged_mfa_coverage_pct": 100,
    "privileged_phishing_resistant_pct": 62.5,
    "standing_privileged_users_count": 2
  },
  "lifecycle": {
    "suspended_pct": 1.8,
    "archived_pct": 0.6
  },
  "policy": {
    "mfa_required": true,
    "mfa_required_coverage_pct": 100,
    "session_lifetime_max_min": 480,
    "idle_timeout_max_min": 30,
    "legacy_auth_blocked": true,
    "phishing_resistant_required_for_privileged": true
  },
  "device_access": {
    "managed_device_required": true,
    "managed_device_required_for_admins": true
  }
}
```

## 4. Field Mapping by Provider

| Normalized Field | Okta | Google Workspace | Ping Identity | Microsoft Entra ID |
|-----------------|------|------------------|---------------|-------------------|
| `user_security.mfa_coverage_pct` | Users with MFA factors enrolled | Users protected by 2SV | Users with MFA configured | Users with MFA registered |
| `user_security.mfa_phishing_resistant_pct` | Users with WebAuthn factor | Users with passkeys or security keys | Users with FIDO2 | Users with FIDO2 security key |
| `user_security.inactive_pct` | `lastLogin` > 90 days | `lastLoginTime` > 90 days | `lastSignOn` > 90 days | `signInActivity.lastSignInDateTime` > 90 days |
| `user_security.weak_password_pct` | Password health or expired/weak signals if available | Google weak password signal | Password health if available | Password protection risk or password policy health if available |
| `app_security.provisioning_enabled_pct` | Apps with lifecycle provisioning | Not generally exposed | Apps with provisioning connectors | Enterprise apps with SCIM or provisioning enabled |
| `app_security.deprovisioning_enabled_pct` | Apps with automated deprovisioning | Not generally exposed | Apps with deprovisioning workflows | Enterprise apps with automatic deprovisioning |
| `privileged_access.super_admin_count` | Super admins or org admins | Super admins | Highest-privilege admins | Global admins or equivalent privileged roles |
| `privileged_access.privileged_mfa_coverage_pct` | Admins with MFA coverage | Privileged users with 2SV enforced | Admins with MFA coverage | Privileged role holders with MFA |
| `policy.mfa_required_coverage_pct` | Users covered by MFA sign-on policy | Users with 2SV enforced | Users covered by MFA policy | Users covered by Conditional Access MFA |
| `policy.legacy_auth_blocked` | Legacy auth restrictions or password-only restrictions | IMAP/POP and less secure app controls | Legacy auth restrictions | Exchange/tenant legacy auth disabled |
| `device_access.managed_device_required` | Device trust / device assurance policy coverage | Context-aware access and endpoint verification | Device trust policy | Conditional Access compliant device requirement |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| MFA coverage | `user_security.mfa_coverage_pct`, `policy.mfa_required_coverage_pct` | `>= 95` and `>= 95` |
| MFA for privileged accounts | `privileged_access.privileged_mfa_coverage_pct` | `== 100` |
| Phishing-resistant MFA | `user_security.mfa_phishing_resistant_pct`, `privileged_access.privileged_phishing_resistant_pct`, `policy.phishing_resistant_required_for_privileged` | `>= 50`, `== 100`, `== true` |
| Privileged access minimization | `privileged_access.super_admin_count`, `privileged_access.standing_privileged_users_count` | `<= 4`, `== 0` |
| Joiner / mover / leaver automation | `app_security.deprovisioning_enabled_pct`, `user_security.inactive_pct`, `lifecycle.suspended_pct` | `>= 90`, `<= 10`, field presence indicates tracking |
| Legacy authentication blocked | `policy.legacy_auth_blocked` | `== true` |
| Session timeout | `policy.session_lifetime_max_min` | `<= 480` (8 hours) |
| Idle timeout | `policy.idle_timeout_max_min` | `<= 30` |
| Password hygiene | `user_security.weak_password_pct`, `user_security.password_policy_noncompliant_pct` | `<= 5`, `<= 1` |
| Managed device enforcement | `device_access.managed_device_required`, `device_access.managed_device_required_for_admins` | `== true`, `== true` |
| Third-party app governance | `app_security.authorized_third_party_apps_count` | Trend over time or below organization threshold |
