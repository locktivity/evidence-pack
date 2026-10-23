# TLS Posture Schema

**Type Identifier:** `evidencepack/tls@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for the TLS posture of a list of endpoints. For each endpoint it holds the TLS versions the endpoint accepts, the certificate it presents, and an assessment of them, and a summary covers every endpoint.

Only `schema_version`, `collected_at`, `endpoints`, and each endpoint's `host` and `port` are required. A `certificate` that is present must have `not_before`, `not_after`, and `fingerprint_sha256`. Fields the schema does not declare are not allowed. A value the collector could not read is absent or `null`.

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "evidencepack/tls@v1",
  "title": "TLS posture",
  "type": "object",
  "additionalProperties": false,
  "required": ["schema_version", "collected_at", "endpoints"],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "1.0.0"
    },
    "collected_at": {
      "type": "string",
      "format": "date-time"
    },
    "endpoints": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/endpoint_result"
      }
    },
    "summary": {
      "$ref": "#/$defs/summary"
    },
    "diagnostics": {
      "$ref": "#/$defs/diagnostics"
    }
  },
  "$defs": {
    "endpoint_result": {
      "type": "object",
      "additionalProperties": false,
      "required": ["host", "port"],
      "properties": {
        "host": {
          "type": "string"
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        },
        "reachable": {
          "type": ["boolean", "null"]
        },
        "tls": {
          "$ref": "#/$defs/tls_details"
        },
        "certificate": {
          "$ref": "#/$defs/certificate"
        },
        "assessment": {
          "$ref": "#/$defs/assessment"
        },
        "errors": {
          "type": "array",
          "items": {
            "type": "string"
          }
        }
      }
    },
    "tls_details": {
      "type": ["object", "null"],
      "additionalProperties": false,
      "properties": {
        "negotiated_version": {
          "type": "string"
        },
        "negotiated_cipher_suite": {
          "type": "string"
        },
        "protocol_support": {
          "$ref": "#/$defs/protocol_support"
        }
      }
    },
    "protocol_support": {
      "type": ["object", "null"],
      "additionalProperties": false,
      "properties": {
        "tls_1_0": {
          "type": ["boolean", "null"]
        },
        "tls_1_1": {
          "type": ["boolean", "null"]
        },
        "tls_1_2": {
          "type": ["boolean", "null"]
        },
        "tls_1_3": {
          "type": ["boolean", "null"]
        }
      }
    },
    "certificate": {
      "type": ["object", "null"],
      "additionalProperties": false,
      "required": ["not_before", "not_after", "fingerprint_sha256"],
      "properties": {
        "subject_common_name": {
          "type": "string"
        },
        "issuer_common_name": {
          "type": "string"
        },
        "serial_number": {
          "type": "string"
        },
        "not_before": {
          "type": "string",
          "format": "date-time"
        },
        "not_after": {
          "type": "string",
          "format": "date-time"
        },
        "expires_in_days": {
          "type": ["integer", "null"]
        },
        "dns_names": {
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "ip_addresses": {
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "fingerprint_sha256": {
          "type": "string",
          "pattern": "^[A-F0-9]{64}$"
        },
        "signature_algorithm": {
          "type": "string"
        },
        "public_key_algorithm": {
          "type": "string"
        },
        "chain_length": {
          "type": ["integer", "null"],
          "minimum": 1
        },
        "trusted_chain": {
          "type": ["boolean", "null"]
        },
        "hostname_valid": {
          "type": ["boolean", "null"]
        }
      }
    },
    "assessment": {
      "type": ["object", "null"],
      "additionalProperties": false,
      "properties": {
        "certificate_valid": {
          "type": ["boolean", "null"]
        },
        "hostname_valid": {
          "type": ["boolean", "null"]
        },
        "expired": {
          "type": ["boolean", "null"]
        },
        "expiring_soon": {
          "type": ["boolean", "null"]
        },
        "modern_protocol": {
          "type": ["boolean", "null"]
        },
        "legacy_protocol_supported": {
          "type": ["boolean", "null"]
        },
        "issues": {
          "type": "array",
          "items": {
            "type": "string"
          }
        }
      }
    },
    "summary": {
      "type": ["object", "null"],
      "additionalProperties": false,
      "properties": {
        "total_endpoints": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "reachable_count": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "reachable_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100
        },
        "certificate_valid_count": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "certificate_valid_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100
        },
        "hostname_valid_count": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "hostname_valid_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100
        },
        "modern_protocol_count": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "modern_protocol_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100
        },
        "expiring_soon_count": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "expiring_soon_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100
        },
        "legacy_protocol_supported_count": {
          "type": ["integer", "null"],
          "minimum": 0
        },
        "legacy_protocol_supported_pct": {
          "type": ["integer", "null"],
          "minimum": 0,
          "maximum": 100
        },
        "all_certificates_valid": {
          "type": ["boolean", "null"]
        },
        "all_modern_protocol": {
          "type": ["boolean", "null"]
        }
      }
    },
    "diagnostics": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "endpoint_errors": {
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "warnings": {
          "type": "array",
          "items": {
            "type": "string"
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
  "endpoints": [
    {
      "host": "www.example.com",
      "port": 443,
      "reachable": true,
      "tls": {
        "negotiated_version": "TLS 1.3",
        "negotiated_cipher_suite": "TLS_AES_128_GCM_SHA256",
        "protocol_support": {
          "tls_1_0": false,
          "tls_1_1": false,
          "tls_1_2": true,
          "tls_1_3": true
        }
      },
      "certificate": {
        "subject_common_name": "www.example.com",
        "issuer_common_name": "Example Issuing CA",
        "serial_number": "271828182845904523536028747135266249775",
        "not_before": "2026-01-10T00:00:00Z",
        "not_after": "2026-04-10T23:59:59Z",
        "expires_in_days": 23,
        "dns_names": [
          "www.example.com",
          "example.com"
        ],
        "ip_addresses": [],
        "fingerprint_sha256": "168C0D642942111032099DBB3EAB4E224A7D2A3E12B4D79B423ACAFA3D3705B3",
        "signature_algorithm": "SHA256-RSA",
        "public_key_algorithm": "RSA",
        "chain_length": 2,
        "trusted_chain": true,
        "hostname_valid": true
      },
      "assessment": {
        "certificate_valid": true,
        "hostname_valid": true,
        "expired": false,
        "expiring_soon": true,
        "modern_protocol": true,
        "legacy_protocol_supported": false,
        "issues": [
          "Certificate expires within 30 days"
        ]
      }
    },
    {
      "host": "legacy.example.com",
      "port": 8443,
      "reachable": true,
      "tls": {
        "negotiated_version": "TLS 1.2",
        "negotiated_cipher_suite": "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
        "protocol_support": {
          "tls_1_0": true,
          "tls_1_1": true,
          "tls_1_2": true,
          "tls_1_3": false
        }
      },
      "certificate": {
        "subject_common_name": "legacy.example.com",
        "issuer_common_name": "Example Issuing CA",
        "serial_number": "314159265358979323846264338327950288419",
        "not_before": "2025-12-01T00:00:00Z",
        "not_after": "2026-12-01T23:59:59Z",
        "expires_in_days": 258,
        "dns_names": [
          "legacy.example.com"
        ],
        "ip_addresses": [],
        "fingerprint_sha256": "D3F9E6B4D595050B6164C734E3F36F4D79C06E80FBAC2AB8600FBD7E98395697",
        "signature_algorithm": "SHA256-RSA",
        "public_key_algorithm": "RSA",
        "chain_length": 2,
        "trusted_chain": true,
        "hostname_valid": true
      },
      "assessment": {
        "certificate_valid": true,
        "hostname_valid": true,
        "expired": false,
        "expiring_soon": false,
        "modern_protocol": true,
        "legacy_protocol_supported": true,
        "issues": [
          "TLS 1.0 or TLS 1.1 is supported"
        ]
      }
    }
  ],
  "summary": {
    "total_endpoints": 2,
    "reachable_count": 2,
    "reachable_pct": 100,
    "certificate_valid_count": 2,
    "certificate_valid_pct": 100,
    "hostname_valid_count": 2,
    "hostname_valid_pct": 100,
    "modern_protocol_count": 2,
    "modern_protocol_pct": 100,
    "expiring_soon_count": 1,
    "expiring_soon_pct": 50,
    "legacy_protocol_supported_count": 1,
    "legacy_protocol_supported_pct": 50,
    "all_certificates_valid": true,
    "all_modern_protocol": true
  }
}
```

## 4. Field Sources

| Normalized Field | Source |
|-----------------|--------|
| `endpoints[].reachable` | The endpoint completed a handshake that offered TLS 1.2 and TLS 1.3 |
| `endpoints[].tls.negotiated_version` | The version that handshake settled on, such as `TLS 1.3` |
| `endpoints[].tls.protocol_support` | Whether the endpoint accepts each TLS version, from that handshake or from one that offered only that version |
| `endpoints[].certificate` | The leaf certificate the endpoint presented in that handshake |
| `endpoints[].certificate.fingerprint_sha256` | SHA-256 digest of the certificate's DER encoding, in uppercase hex |
| `endpoints[].certificate.trusted_chain` | The certificates the endpoint sent lead to a trusted root. Expiry and the host name are judged on their own |
| `endpoints[].assessment.certificate_valid` | The chain is trusted, the certificate is valid for the host, and the check falls inside its validity window |
| `endpoints[].assessment.expiring_soon` | The certificate has not expired and expires within 30 days |
| `endpoints[].assessment.modern_protocol` | The endpoint accepts TLS 1.2 or TLS 1.3 |
| `endpoints[].assessment.legacy_protocol_supported` | The endpoint accepts TLS 1.0 or TLS 1.1 |
| `summary.certificate_valid_pct` | `summary.certificate_valid_count` over `summary.total_endpoints`, rounded down. The other percentages in `summary` work the same way |

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Encryption in transit | `summary.all_modern_protocol` | `== true` |
| Legacy protocols disabled | `summary.legacy_protocol_supported_count` | `== 0` |
| Valid certificates | `summary.all_certificates_valid` | `== true` |
| Certificate renewal | `summary.expiring_soon_count` | `== 0` |
