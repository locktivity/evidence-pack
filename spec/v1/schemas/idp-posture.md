# Identity Provider Posture Schema

**Type Identifier:** `evidencepack/idp-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for identity provider security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (Okta, Google Workspace, Microsoft Entra ID, JumpCloud, Ping Identity, etc.).

Only `schema_version`, `collected_at`, `provider`, and `org_domain` are required. The posture sections are optional and SHOULD be emitted when the provider exposes them. A value the collector could not read is absent or `null`.

Each percentage has its count beside it. The `user_security` percentages and `policy.mfa_required_coverage_pct` are shares of `user_security.user_count`. The `app_security` percentages are shares of `app_count`, and the privileged percentages are shares of `privileged_users_count`. The `lifecycle` percentages are shares of `account_count`, which includes suspended and archived accounts. Percentages are whole numbers. A percentage where more is better, such as `mfa_coverage_pct`, rounds down. One where more is worse, such as `inactive_pct`, rounds up. The counts are optional. When they are absent, the `_present` booleans show whether a section had anyone to count. When one is false, that section's percentages are `null`. `policy.mfa_required_coverage_pct` is also `null` when `user_security.users_present` is false.

An artifact is collected at the trust, audit, or internal level. The fields whose names end in `_count` are written at the audit and internal levels only. Every other field is written at every level, including the `_present` booleans and the percentages. The level is not a field of the artifact. The example in Section 3 is an audit-level artifact.

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider", "org_domain"],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "1.0.0",
      "description": "Version of this schema, always 1.0.0."
    },
    "collected_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp when data was collected"
    },
    "provider": {
      "type": "string",
      "enum": ["okta", "ping", "entra", "google_workspace", "auth0", "onelogin", "jumpcloud", "other"],
      "description": "Identity provider vendor"
    },
    "org_domain": {
      "type": "string",
      "description": "Organization domain or tenant identifier"
    },
    "user_security": {
      "type": ["object", "null"],
      "properties": {
        "users_present": {
          "type": ["boolean", "null"],
          "description": "Whether the provider has any users the user_security percentages count. When false, those percentages are null"
        },
        "user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Accounts that can sign in. The user_security percentages and policy.mfa_required_coverage_pct count these"
        },
        "mfa_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Users with any MFA factor enrolled"
        },
        "phishing_resistant_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Users with phishing-resistant MFA (WebAuthn or FIDO2) enrolled"
        },
        "inactive_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Users inactive for 90 days or more"
        },
        "locked_out_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Users currently locked out"
        },
        "weak_password_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Users with weak passwords"
        },
        "password_policy_noncompliant_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Users whose passwords do not meet current password policy requirements"
        },
        "mfa_coverage_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users with any MFA factor enrolled"
        },
        "mfa_phishing_resistant_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users with phishing-resistant MFA (WebAuthn/FIDO2)"
        },
        "inactive_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users inactive for 90+ days"
        },
        "locked_out_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users currently locked out"
        },
        "weak_password_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users with weak passwords"
        },
        "password_policy_noncompliant_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users whose passwords do not meet current password policy requirements"
        }
      },
      "description": "Percentages of the provider's users with MFA, with phishing-resistant MFA, inactive, locked out, and with weak or noncompliant passwords."
    },
    "app_security": {
      "type": ["object", "null"],
      "properties": {
        "apps_present": {
          "type": ["boolean", "null"],
          "description": "Whether the provider has any applications the app_security percentages count. When false, those percentages are null"
        },
        "app_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Applications the app_security percentages count"
        },
        "sso_app_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Applications using federated SSO (SAML or OIDC)"
        },
        "provisioning_app_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Applications with automated provisioning"
        },
        "deprovisioning_app_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Applications with automated deprovisioning"
        },
        "sso_coverage_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of applications using federated SSO (SAML/OIDC)"
        },
        "provisioning_enabled_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of applications with automated provisioning"
        },
        "deprovisioning_enabled_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of applications with automated deprovisioning"
        },
        "authorized_third_party_apps_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Count of authorized third-party applications or OAuth integrations"
        }
      },
      "description": "How the provider's apps sign users in and provision them, and how many third-party apps users have authorized."
    },
    "privileged_access": {
      "type": ["object", "null"],
      "properties": {
        "privileged_users_present": {
          "type": ["boolean", "null"],
          "description": "Whether the provider has any users with privileged administrative access. When false, the privileged percentages are null"
        },
        "privileged_users_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Count of users with privileged administrative access"
        },
        "super_admin_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Count of highest-privilege administrative accounts (for example Global Admin or Super Admin)"
        },
        "privileged_mfa_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Privileged users with MFA enabled or enforced"
        },
        "privileged_phishing_resistant_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Privileged users with phishing-resistant MFA"
        },
        "privileged_mfa_coverage_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of privileged users with MFA enabled or enforced"
        },
        "privileged_phishing_resistant_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of privileged users with phishing-resistant MFA"
        },
        "standing_privileged_users_present": {
          "type": ["boolean", "null"],
          "description": "Whether any user holds standing privileged access, as standing_privileged_users_count defines it"
        },
        "standing_privileged_users_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Count of users with standing privileged access instead of just-in-time or eligible access"
        }
      },
      "description": "Counts of privileged users and the share of them with MFA and with phishing-resistant MFA."
    },
    "lifecycle": {
      "type": ["object", "null"],
      "properties": {
        "accounts_present": {
          "type": ["boolean", "null"],
          "description": "Whether the provider has any accounts, in any state. When false, the lifecycle percentages are null"
        },
        "account_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Accounts in every state, including suspended and archived accounts. The lifecycle percentages count these"
        },
        "suspended_account_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Accounts in a suspended or disabled state"
        },
        "archived_account_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Accounts in an archived, retained, or soft-deleted state"
        },
        "suspended_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of accounts in suspended or disabled state"
        },
        "archived_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of accounts in archived, retained, or soft-deleted state"
        }
      },
      "description": "Percentages of users suspended and archived."
    },
    "policy": {
      "type": ["object", "null"],
      "properties": {
        "mfa_required": {
          "type": ["boolean", "null"],
          "description": "Whether the tenant enforces MFA for all covered users"
        },
        "mfa_required_user_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Users covered by policies that require MFA, out of user_security.user_count"
        },
        "mfa_required_coverage_pct": {
          "type": ["number", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of users covered by policies that require MFA"
        },
        "session_lifetime_max_min": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Maximum session lifetime across all policies (minutes)"
        },
        "idle_timeout_max_min": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Maximum idle timeout across all policies (minutes)"
        },
        "legacy_auth_blocked": {
          "type": ["boolean", "null"],
          "description": "Whether legacy authentication protocols are blocked"
        },
        "phishing_resistant_required_for_privileged": {
          "type": ["boolean", "null"],
          "description": "Whether phishing-resistant authentication is required for privileged users"
        }
      },
      "description": "Sign-in policy settings: MFA requirements, session limits, and legacy authentication."
    },
    "device_access": {
      "type": ["object", "null"],
      "properties": {
        "managed_device_required": {
          "type": ["boolean", "null"],
          "description": "Whether access to core corporate resources requires a managed or compliant device"
        },
        "managed_device_required_for_admins": {
          "type": ["boolean", "null"],
          "description": "Whether privileged access requires a managed or compliant device"
        }
      },
      "description": "Whether sign-in policies require a managed device, for all users and for administrators."
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
    "users_present": true,
    "user_count": 400,
    "mfa_user_count": 394,
    "phishing_resistant_user_count": 180,
    "inactive_user_count": 9,
    "locked_out_user_count": 2,
    "weak_password_user_count": 12,
    "password_policy_noncompliant_user_count": 5,
    "mfa_coverage_pct": 98,
    "mfa_phishing_resistant_pct": 45,
    "inactive_pct": 3,
    "locked_out_pct": 1,
    "weak_password_pct": 3,
    "password_policy_noncompliant_pct": 2
  },
  "app_security": {
    "apps_present": true,
    "app_count": 40,
    "sso_app_count": 34,
    "provisioning_app_count": 24,
    "deprovisioning_app_count": 23,
    "sso_coverage_pct": 85,
    "provisioning_enabled_pct": 60,
    "deprovisioning_enabled_pct": 57,
    "authorized_third_party_apps_count": 41
  },
  "privileged_access": {
    "privileged_users_present": true,
    "privileged_users_count": 8,
    "super_admin_count": 3,
    "privileged_mfa_user_count": 8,
    "privileged_phishing_resistant_user_count": 5,
    "privileged_mfa_coverage_pct": 100,
    "privileged_phishing_resistant_pct": 62,
    "standing_privileged_users_present": true,
    "standing_privileged_users_count": 2
  },
  "lifecycle": {
    "accounts_present": true,
    "account_count": 412,
    "suspended_account_count": 8,
    "archived_account_count": 4,
    "suspended_pct": 2,
    "archived_pct": 1
  },
  "policy": {
    "mfa_required": true,
    "mfa_required_user_count": 400,
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

| Normalized Field | Okta | Google Workspace | Microsoft Entra ID | JumpCloud |
|-----------------|------|------------------|-------------------|-----------|
| `user_security.mfa_coverage_pct` | Users with MFA factors enrolled | Users protected by 2SV or a passkey | Members with MFA registered | Active users with any MFA factor enrolled |
| `user_security.mfa_phishing_resistant_pct` | Users with a WebAuthn factor | Users with passkeys or security keys | Members with FIDO2, Windows Hello for Business, passkey, or certificate methods registered | Users with a WebAuthn device enrolled |
| `user_security.inactive_pct` | `lastLogin` > 90 days | `lastLoginTime` > 90 days | `signInActivity.lastSignInDateTime` > 90 days | Not collected |
| `user_security.locked_out_pct` | Users in the `LOCKED_OUT` state | Users with a locked account | Not collected | Locked users |
| `user_security.weak_password_pct` | Not collected | Google weak password signal | Not collected | Not collected |
| `user_security.password_policy_noncompliant_pct` | Not collected | Passwords shorter than the length policy | Not collected | Not collected |
| `app_security.sso_coverage_pct` | Apps signing in with SAML, OIDC, or WS-Federation | Not collected | Enabled enterprise apps with federated single sign-on | Applications with SSO configured |
| `app_security.provisioning_enabled_pct` | Apps with lifecycle provisioning | Not collected | Not collected | Not collected |
| `app_security.deprovisioning_enabled_pct` | Apps with automated deprovisioning | Not collected | Not collected | Not collected |
| `app_security.authorized_third_party_apps_count` | Not collected | Third-party apps users have authorized | Not collected | Not collected |
| `privileged_access.super_admin_count` | Super admins | Super admins | Global Administrators | Administrators with the Administrator With Billing role |
| `privileged_access.privileged_mfa_coverage_pct` | Admins with MFA | Privileged users enrolled in 2SV | Privileged role holders with MFA | 100 when every administrator has TOTP enrolled, otherwise absent |
| `privileged_access.privileged_phishing_resistant_pct` | Admins with a WebAuthn factor | Not collected | Not collected | Not collected |
| `privileged_access.standing_privileged_users_count` | Users holding an admin role | Users holding an admin role | Not collected | Not collected |
| `policy.mfa_required_coverage_pct` | Users covered by an MFA sign-on policy | Users with 2SV enforced | Users covered by Conditional Access MFA | Not collected |
| `policy.session_lifetime_max_min`, `policy.idle_timeout_max_min` | Longest session and idle limits across sign-on policies | Not collected | Not collected | Not collected |
| `policy.legacy_auth_blocked` | Not collected | Less secure app access, which Google no longer allows | Conditional Access policy blocking legacy authentication clients | Not collected |
| `lifecycle.suspended_pct` | Users in the `SUSPENDED` state | Suspended users | Disabled accounts | Suspended users |
| `lifecycle.archived_pct` | Not collected | Archived users | Not collected | Not collected |
| `device_access.managed_device_required` | Sign-on policy requiring a managed device | Context-aware access requiring a managed device | Not collected | Not collected |

Each count beside a percentage comes from the same source as the percentage.

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

`privileged_access.super_admin_count`, `privileged_access.standing_privileged_users_count`, and `app_security.authorized_third_party_apps_count` are written at the audit and internal levels only. A trust-level artifact has `privileged_access.standing_privileged_users_present` in place of the standing count.
