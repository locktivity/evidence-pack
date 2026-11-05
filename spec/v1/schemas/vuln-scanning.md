# Vulnerability Scanning Schema

**Type Identifier:** `evidencepack/vuln-scanning@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for vulnerability scanning data from one system. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (AWS, GitHub, etc.).

`scanners` has one entry for each scanner and category. A scanner that covers several kinds of resource has an entry for each kind, so a scanner of hosts and container images has two entries. Each entry holds the scanner's coverage and its open findings by severity.

Only `schema_version`, `collected_at`, `provider`, `scanners`, and each entry's `scanner` and `category` are required. Every metric is optional. A value the collector could not read is absent or `null`.

`coverage_pct` is `scanned_resource_count` out of `resource_count`. It is a whole number and rounds down. When `resources_present` is false, `coverage_pct` is `null`.

A scanner turned on per resource, such as per repository, is `enabled` when it is on for at least one resource. `coverage_pct` gives the share of resources it covers.

`open_critical_count`, `open_high_count`, `open_medium_count`, and `open_low_count` are the findings open at each severity. A finding is overdue when it is critical and open more than 30 days, or high and open more than 90 days. `overdue_critical_count` and `overdue_high_count` count the overdue findings.

An artifact is collected at the trust, audit, or internal level. `scopes` and the fields whose names end in `_count` are written at the audit and internal levels only. Every other field is written at every level, including `coverage_pct` and the `_present` booleans. At the trust level, the critical, high, and overdue counts are left out. `open_critical_present`, `open_high_present`, and `overdue_present` are true when there are any. The level is not a field of the artifact. The example in Section 3 is an audit-level artifact.

When a system has several accounts or subscriptions, `scanners` holds totals across all of them. `scopes` lists each account or subscription separately (Section 6).

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "description": "Normalized vulnerability scanning evidence for one system, such as AWS or GitHub. Each entry in scanners gives one scanner's coverage and its open findings by severity. A finding is overdue when it is critical and open more than 30 days, or high and open more than 90 days. When a system has several accounts, scanners holds totals across all of them, and scopes lists each account at the audit level. A *_present flag is then true when it is true in any scope. Every metric is optional. Below the audit level, counts are left out, and the *_present flags are written instead",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider", "scanners"],
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
      "description": "The system the scanners run in, such as aws or github"
    },
    "scanners": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/scanner"
      },
      "description": "One entry per scanner and category. In one scope, a scanner turned on per resource, such as per repository, is enabled when it is on for at least one resource, and coverage_pct gives the share it covers. When the system has scopes, the top-level enabled is true only when the scanner is enabled in every scope"
    },
    "scopes": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/scope"
      },
      "description": "One entry per account or subscription. Each entry lists that account's scanners"
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
    "scanner": {
      "type": "object",
      "required": ["scanner", "category"],
      "properties": {
        "scanner": {
          "type": "string",
          "pattern": "^[a-z0-9_]+$",
          "description": "A fixed name for the scanner, such as inspector, dependabot, code_scanning, or secret_scanning"
        },
        "category": {
          "enum": ["hosts", "containers", "functions", "dependencies", "code", "secrets"],
          "description": "The kind of resource the entry covers. A scanner that covers several kinds has an entry for each"
        },
        "enabled": {
          "type": ["boolean", "null"],
          "description": "Whether the scanner is turned on for this category. A scanner turned on per resource, such as per repository, is on when it covers at least one resource"
        },
        "resources_present": {
          "type": ["boolean", "null"],
          "description": "Whether there are resources the scanner could cover. When false, coverage_pct has none to count"
        },
        "resource_count": {
          "$ref": "#/$defs/count",
          "description": "Resources the scanner could cover"
        },
        "scanned_resource_count": {
          "$ref": "#/$defs/count",
          "description": "Resources counted in resource_count that the scanner covers"
        },
        "coverage_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of resources the scanner covers. Null when there are none"
        },
        "open_critical_present": {
          "type": ["boolean", "null"],
          "description": "Whether the scanner has open critical findings"
        },
        "open_high_present": {
          "type": ["boolean", "null"],
          "description": "Whether the scanner has open high findings"
        },
        "overdue_present": {
          "type": ["boolean", "null"],
          "description": "Whether the scanner has overdue findings"
        },
        "open_critical_count": {
          "$ref": "#/$defs/count",
          "description": "Open critical findings"
        },
        "open_high_count": {
          "$ref": "#/$defs/count",
          "description": "Open high findings"
        },
        "open_medium_count": {
          "$ref": "#/$defs/count",
          "description": "Open medium findings"
        },
        "open_low_count": {
          "$ref": "#/$defs/count",
          "description": "Open low findings"
        },
        "overdue_critical_count": {
          "$ref": "#/$defs/count",
          "description": "Critical findings open more than 30 days"
        },
        "overdue_high_count": {
          "$ref": "#/$defs/count",
          "description": "High findings open more than 90 days"
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
        "kind": {
          "enum": ["account", "subscription"]
        },
        "scanners": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/scanner"
          },
          "description": "The scanners for this scope alone, with the same fields"
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
  "scanners": [
    {
      "scanner": "dependabot",
      "category": "dependencies",
      "enabled": true,
      "resources_present": true,
      "resource_count": 48,
      "scanned_resource_count": 46,
      "coverage_pct": 95,
      "open_critical_present": true,
      "open_high_present": true,
      "overdue_present": true,
      "open_critical_count": 2,
      "open_high_count": 9,
      "open_medium_count": 31,
      "open_low_count": 12,
      "overdue_critical_count": 1,
      "overdue_high_count": 3
    },
    {
      "scanner": "code_scanning",
      "category": "code",
      "enabled": true,
      "resources_present": true,
      "resource_count": 48,
      "scanned_resource_count": 29,
      "coverage_pct": 60,
      "open_critical_present": false,
      "open_high_present": true,
      "overdue_present": false,
      "open_critical_count": 0,
      "open_high_count": 4,
      "open_medium_count": 17,
      "open_low_count": 25,
      "overdue_critical_count": 0,
      "overdue_high_count": 0
    },
    {
      "scanner": "secret_scanning",
      "category": "secrets",
      "enabled": true,
      "resources_present": true,
      "resource_count": 48,
      "scanned_resource_count": 48,
      "coverage_pct": 100
    }
  ]
}
```

## 4. Example Scanners

`scanner` is a fixed name in lowercase letters, digits, and underscores. The table lists common scanners. The last column names the resources that `resource_count` counts.

| `provider` | `scanner` | `category` | Resources |
|------------|-----------|------------|-----------|
| `aws` | `inspector` | `hosts` | EC2 instances |
| `aws` | `inspector` | `containers` | Container images in Amazon ECR |
| `aws` | `inspector` | `functions` | Lambda functions |
| `github` | `dependabot` | `dependencies` | Repositories |
| `github` | `code_scanning` | `code` | Repositories |
| `github` | `secret_scanning` | `secrets` | Repositories |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Vulnerability scanning enabled | `scanners[].enabled` | `== true` |
| Scan coverage | `scanners[].coverage_pct` | `>= 90` |
| No open critical vulnerabilities | `scanners[].open_critical_present` | `== false` |
| Vulnerability remediation | `scanners[].overdue_present` | `== false` |

A condition that checks every entry uses a path such as `$.scanners[*].enabled` with `cardinality` set to `all`. A pack can hold a vuln-scanning artifact for each system. A clause that should hold on every system sets `artifacts: all` ([profiles.md](../profiles.md) Section 4.4).

## 6. Multiple Accounts

When a system has several accounts or subscriptions, such as the accounts in an AWS organization, `scanners` holds totals across all of them. `scopes` has one entry for each account or subscription. Each entry has an `id` and a `kind` of `account` or `subscription`. Its `scanners` list holds the entries for that account alone. A top-level `enabled` is true only when the scanner is enabled in every scope. A `_present` flag is true when it is true in any scope.

`scopes` is written at the audit and internal levels only, so a trust-level artifact has the totals alone.
