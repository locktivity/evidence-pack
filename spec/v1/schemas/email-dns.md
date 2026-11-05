# Email DNS Posture Schema

**Type Identifier:** `evidencepack/email-dns@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for email authentication posture read from public DNS. For each domain it holds the MX, SPF, DMARC, and DKIM records and an assessment of them, and a summary covers every domain.

Only `schema_version`, `collected_at`, `domains`, and each domain's `domain` are required. A domain's `spf` or `dmarc` is `null` when the domain publishes no such record, and absent when its lookup failed. Any other value the collector could not read is absent or `null`.

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/locktivity/epack-collector-email-dns/docs/schema/v1.0.0.json",
  "title": "Email DNS Posture",
  "description": "Email DNS security posture for a list of domains",
  "type": "object",
  "required": ["schema_version", "collected_at", "domains"],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "1.0.0"
    },
    "collected_at": {
      "type": "string",
      "format": "date-time"
    },
    "domains": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/DomainResult"
      }
    },
    "summary": {
      "$ref": "#/$defs/Summary"
    },
    "diagnostics": {
      "$ref": "#/$defs/Diagnostics"
    }
  },
  "$defs": {
    "DomainResult": {
      "type": "object",
      "required": ["domain"],
      "properties": {
        "domain": {
          "type": "string",
          "description": "The domain name"
        },
        "mx": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/MXRecord"
          }
        },
        "spf": {
          "oneOf": [
            { "$ref": "#/$defs/SPFRecord" },
            { "type": "null" }
          ]
        },
        "dmarc": {
          "oneOf": [
            { "$ref": "#/$defs/DMARCRecord" },
            { "type": "null" }
          ]
        },
        "dkim": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/DKIMRecord"
          }
        },
        "assessment": {
          "$ref": "#/$defs/Assessment"
        },
        "errors": {
          "type": "array",
          "items": {
            "type": "string"
          }
        }
      }
    },
    "MXRecord": {
      "type": "object",
      "required": ["host", "priority"],
      "properties": {
        "host": {
          "type": "string",
          "description": "Mail server hostname"
        },
        "priority": {
          "type": "integer",
          "minimum": 0,
          "maximum": 65535,
          "description": "MX priority (lower is higher priority)"
        }
      }
    },
    "SPFRecord": {
      "type": "object",
      "required": ["record"],
      "properties": {
        "record": {
          "type": "string",
          "description": "The SPF TXT record value"
        },
        "valid": {
          "type": ["boolean", "null"],
          "description": "Whether the record is valid"
        }
      }
    },
    "DMARCRecord": {
      "type": "object",
      "required": ["record"],
      "properties": {
        "record": {
          "type": "string",
          "description": "The DMARC TXT record value"
        },
        "policy": {
          "type": "string",
          "enum": ["none", "quarantine", "reject", "unknown"],
          "description": "The policy the record requests for the domain, from its p tag. For a record without a valid p tag, this is none when the record lists a valid rua URI, as RFC 9989 specifies. It is unknown when no DMARC processing applies to the record"
        },
        "subdomain_policy": {
          "type": "string",
          "enum": ["none", "quarantine", "reject", "unknown", ""],
          "description": "The policy the record requests for existing subdomains, from its sp tag. When the record has no sp tag, this is the same as policy"
        },
        "pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "The pct tag defined in RFC 7489. RFC 9989 drops it, and receivers ignore it. It is recorded for reference and does not affect any other value"
        },
        "rua": {
          "type": "string",
          "description": "The aggregate report URIs from the rua tag, as written"
        },
        "ruf": {
          "type": "string",
          "description": "The failure report URIs from the ruf tag, as written"
        },
        "valid": {
          "type": ["boolean", "null"],
          "description": "Whether the record is valid"
        }
      }
    },
    "DKIMRecord": {
      "type": "object",
      "required": ["selector", "record"],
      "properties": {
        "selector": {
          "type": "string",
          "description": "DKIM selector name"
        },
        "record": {
          "type": "string",
          "description": "The DKIM TXT record value"
        },
        "valid": {
          "type": ["boolean", "null"],
          "description": "Whether the record is valid"
        }
      }
    },
    "Assessment": {
      "type": ["object", "null"],
      "properties": {
        "enforced": {
          "type": ["boolean", "null"],
          "description": "True if the DMARC policy that applies to the domain rejects failing mail. That is p=reject on the domain's own record, or a subdomain policy of reject on the record that RFC 9989's DNS tree walk finds above the domain. The subdomain policy is sp, or p when the record has no sp. In either case the record must not set t=y. With t=y, receivers apply the policy one level lower"
        },
        "has_reporting": {
          "type": ["boolean", "null"],
          "description": "True if the DMARC record that applies to the domain lists at least one valid rua URI, such as mailto:dmarc@example.com. An address without a scheme is not a URI, and receivers do not send reports to it"
        },
        "issues": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "List of problems found"
        }
      }
    },
    "Summary": {
      "type": ["object", "null"],
      "properties": {
        "total_domains": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Total number of domains scanned"
        },
        "with_mx_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of domains with at least one MX record"
        },
        "with_mx_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of domains with MX records"
        },
        "with_spf_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of domains with an SPF record, valid or not"
        },
        "with_spf_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of domains with SPF records"
        },
        "with_dmarc_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of domains that a DMARC record applies to, valid or not. The record is the domain's own, or the one RFC 9989's DNS tree walk finds above it"
        },
        "with_dmarc_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of domains that a DMARC record applies to, counted as in with_dmarc_count"
        },
        "with_dkim_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of domains with a DKIM record at one of the common selectors"
        },
        "with_dkim_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of domains with DKIM records"
        },
        "with_all_three_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of domains with MX and SPF records and a DMARC record that applies to them"
        },
        "with_all_three_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of domains with MX and SPF records and a DMARC record that applies to them"
        },
        "enforced_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of domains whose applicable DMARC policy rejects failing mail (assessment.enforced is true)"
        },
        "enforced_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of domains whose applicable DMARC policy rejects failing mail (assessment.enforced is true)"
        },
        "with_reporting_count": {
          "type": ["integer", "null"],
          "minimum": 0,
          "description": "Number of domains whose applicable DMARC record lists a valid rua URI (assessment.has_reporting is true)"
        },
        "with_reporting_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100,
          "description": "Percentage of domains whose applicable DMARC record lists a valid rua URI (assessment.has_reporting is true)"
        },
        "all_enforced": {
          "type": ["boolean", "null"],
          "description": "True if every domain's applicable DMARC policy rejects failing mail (assessment.enforced is true for every domain)"
        }
      }
    },
    "Diagnostics": {
      "type": "object",
      "properties": {
        "domain_errors": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Errors encountered for specific domains"
        },
        "warnings": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Non-fatal warnings"
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
  "domains": [
    {
      "domain": "example.com",
      "mx": [
        {
          "host": "mx1.example.com",
          "priority": 10
        },
        {
          "host": "mx2.example.com",
          "priority": 20
        }
      ],
      "spf": {
        "record": "v=spf1 include:_spf.example.net -all",
        "valid": true
      },
      "dmarc": {
        "record": "v=DMARC1; p=reject; pct=100; rua=mailto:dmarc-reports@example.com",
        "policy": "reject",
        "subdomain_policy": "reject",
        "pct": 100,
        "rua": "mailto:dmarc-reports@example.com",
        "valid": true
      },
      "dkim": [
        {
          "selector": "selector1",
          "record": "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...",
          "valid": true
        }
      ],
      "assessment": {
        "enforced": true,
        "has_reporting": true,
        "issues": []
      }
    },
    {
      "domain": "example.org",
      "mx": [],
      "spf": {
        "record": "v=spf1 -all",
        "valid": true
      },
      "dmarc": {
        "record": "v=DMARC1; p=quarantine; pct=100",
        "policy": "quarantine",
        "subdomain_policy": "quarantine",
        "pct": 100,
        "valid": true
      },
      "dkim": [],
      "assessment": {
        "enforced": false,
        "has_reporting": false,
        "issues": [
          "DMARC policy is 'quarantine' (not reject)",
          "No DMARC aggregate reporting (rua) configured",
          "Subdomain policy is not 'reject'",
          "No DKIM records found for common selectors (custom selectors not checked)"
        ]
      }
    }
  ],
  "summary": {
    "total_domains": 2,
    "with_mx_count": 1,
    "with_mx_pct": 50,
    "with_spf_count": 2,
    "with_spf_pct": 100,
    "with_dmarc_count": 2,
    "with_dmarc_pct": 100,
    "with_dkim_count": 1,
    "with_dkim_pct": 50,
    "with_all_three_count": 1,
    "with_all_three_pct": 50,
    "enforced_count": 1,
    "enforced_pct": 50,
    "with_reporting_count": 1,
    "with_reporting_pct": 50,
    "all_enforced": false
  }
}
```

## 4. Field Sources

| Normalized Field | Source |
|-----------------|--------|
| `domains[].mx` | MX records for the domain, sorted by priority |
| `domains[].spf` | The TXT record at the domain that starts with `v=spf1` |
| `domains[].dmarc` | The TXT record at `_dmarc.<domain>` |
| `domains[].dkim` | TXT records at `<selector>._domainkey.<domain>` for common selectors, such as `selector1`, `selector2`, and `google` |
| `domains[].dmarc.pct` | The `pct` tag, recorded for reference. RFC 9989 drops it, and it does not affect any other value |
| `domains[].assessment.enforced` | The DMARC policy that applies to the domain is `reject`. That is `p=reject` on the domain's own record, or a subdomain policy of `reject` on the record that RFC 9989's DNS tree walk finds above the domain. A record that sets `t=y` does not count |
| `domains[].assessment.has_reporting` | The DMARC record that applies to the domain lists at least one valid `rua` URI, such as `mailto:dmarc@example.com` |
| `summary.total_domains` | Domains scanned, one per entry in `domains` |
| `summary.with_dmarc_pct` | `summary.with_dmarc_count` over `summary.total_domains`, rounded down. The other percentages in `summary` work the same way |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Email spoofing protection | `summary.enforced_pct` | `== 100` |
| DMARC published | `summary.with_dmarc_pct` | `== 100` |
| Sender authentication | `summary.with_spf_pct`, `summary.with_dkim_pct` | `== 100`, `== 100` |
| DMARC reporting | `summary.with_reporting_pct` | `== 100` |
