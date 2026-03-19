# Identity Provider Posture Schema

**Type Identifier:** `evidencepack/idp-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for identity provider security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (Okta, Ping Identity, Microsoft Entra ID, etc.).

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider"],
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
      "enum": ["okta", "ping", "entra", "auth0", "onelogin", "other"],
      "description": "Identity provider vendor"
    },
    "org_domain": {
      "type": "string",
      "description": "Organization domain or tenant identifier"
    },
    "user_security": {
      "type": "object",
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
        }
      }
    },
    "policy": {
      "type": "object",
      "properties": {
        "mfa_required": {
          "type": "boolean",
          "description": "Whether at least one policy requires MFA for all users"
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
    "locked_out_pct": 0.5
  },
  "app_security": {
    "sso_coverage_pct": 85.0,
    "provisioning_enabled_pct": 60.0
  },
  "policy": {
    "mfa_required": true,
    "session_lifetime_max_min": 480,
    "idle_timeout_max_min": 30
  }
}
```

## 4. Field Mapping by Provider

| Normalized Field | Okta | Ping Identity | Microsoft Entra ID |
|-----------------|------|---------------|-------------------|
| `user_security.mfa_coverage_pct` | Users with MFA factors enrolled | Users with MFA configured | Users with MFA registered |
| `user_security.mfa_phishing_resistant_pct` | Users with WebAuthn factor | Users with FIDO2 | Users with FIDO2 security key |
| `user_security.inactive_pct` | `lastLogin` > 90 days | `lastSignOn` > 90 days | `signInActivity.lastSignInDateTime` > 90 days |
| `policy.mfa_required` | Sign-on policy requires MFA | Authentication policy requires MFA | Conditional access requires MFA |
| `policy.session_lifetime_max_min` | Max access token lifetime | Max session duration | Token lifetime policy |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| MFA for privileged accounts | `user_security.mfa_coverage_pct` | `>= 100` |
| Phishing-resistant MFA | `user_security.mfa_phishing_resistant_pct` | `>= 50` |
| Session timeout | `policy.session_lifetime_max_min` | `<= 480` (8 hours) |
| Idle timeout | `policy.idle_timeout_max_min` | `<= 30` |
| Account review | `user_security.inactive_pct` | Field presence indicates tracking |
