# Endpoint Posture Schema

**Type Identifier:** `evidencepack/endpoint-posture@v1`
**Version:** 1.0
**Status:** Draft

## 1. Introduction

This schema defines the normalized format for device management and endpoint security posture data. It provides a vendor-agnostic representation that allows profile requirements to be satisfied by any compliant collector (JumpCloud, Microsoft Intune, Jamf, etc.).

Only `schema_version`, `collected_at`, `provider`, `scope`, and `availability` are required. Every metric is optional. An absent metric has no usable reading, and `availability` lists its path with one of these reasons: `unsupported`, `not_read`, `permission_denied`, `not_configured`, `stale`, `partial`, or `source_error`. The reason is `unsupported` when the vendor never reports the metric, and `not_read` when the vendor reports it but the collector does not read it.

A percentage counts only the devices whose reading passed or failed. Devices whose reading is unknown or does not apply are left out of both sides. `metric_coverage` holds the four counts behind each percentage, keyed by the percentage's path, such as `hardening.disk_encryption_pct`. Percentages are whole numbers and round down. When `metric_coverage` is present, it has an entry for every percentage the artifact holds.

`scope.max_observation_age_seconds` sets how recently a device must have reported. A reading older than that is unknown. When `scope.discovery_complete` is false, the producer could not read every device the provider lists. The artifact then has no `fleet`, `compliance`, `hardening`, `patching`, or `protection` section, and `metric_coverage` is empty. `scope.discovery_complete` is `null` when the producer could not tell.

`compliance.compliant_pct` requires `compliance.basis`, which says whether the verdict is the provider's own or comes from evaluating a baseline. `patching.os_baseline_compliant_pct` requires `patching.baseline_id`.

The optional `devices` array holds a row for each device behind the summary.

An artifact is collected at the trust, audit, or internal level. `organization_id`, `scope.observed_device_count`, the `fleet` counts, `fleet.devices_by_os`, `patching.unsupported_os_count`, `metric_coverage`, and `devices` are written at the audit and internal levels only. Each device's `local_admin_accounts` is written at the internal level only. Every other field is written at every level, including `fleet.devices_present` and the percentages. When `fleet.devices_present` is false, every percentage is `null`. The level is not a field of the artifact. The example in Section 3 is an audit-level artifact.

## 2. Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "description": "Normalized device management and endpoint security evidence. Every metric is optional. An absent metric has no usable reading, and availability records why. Below the audit level, counts that grow with the fleet are left out, and devices_present is written instead. When a device's reading is unknown because a field the collector reads was missing, the percentage still counts the other devices, and it carries a partial gap.",
  "type": "object",
  "required": ["schema_version", "collected_at", "provider", "scope", "availability"],
  "properties": {
    "schema_version": {
      "description": "Version of this schema, always 1.0.0.",
      "type": "string",
      "pattern": "^1\\.\\d+\\.\\d+$"
    },
    "collected_at": {
      "description": "When the collector read the provider, as an ISO 8601 timestamp.",
      "$ref": "#/$defs/timestamp"
    },
    "provider": {
      "description": "The device management or endpoint security product the readings come from.",
      "type": "string",
      "minLength": 1
    },
    "organization_id": {
      "type": "string",
      "description": "The provider's organization identifier"
    },
    "scope": {
      "type": "object",
      "required": ["discovery_complete", "description", "max_observation_age_seconds"],
      "properties": {
        "discovery_complete": {
          "type": ["boolean", "null"],
          "description": "Whether every device the provider lists was read. When false or null, fleet totals and percentages are left out. Null when the collector could not tell"
        },
        "observed_device_count": {
          "$ref": "#/$defs/count",
          "description": "Devices observed within max_observation_age_seconds"
        },
        "description": {
          "type": "string",
          "pattern": "^[a-z0-9_]+$",
          "description": "A fixed name for the scope, such as managed_devices"
        },
        "max_observation_age_seconds": {
          "type": "integer",
          "minimum": 1,
          "description": "How recently a device must have reported to count"
        }
      },
      "additionalProperties": false
    },
    "fleet": {
      "type": "object",
      "properties": {
        "devices_present": {
          "type": ["boolean", "null"],
          "description": "Whether the fleet has any devices in scope. When false, the percentages have no devices to count"
        },
        "device_count": {
          "$ref": "#/$defs/count",
          "description": "Devices in scope, as scope describes them. The fleet percentages count these, and each device row's in_scope says whether it is one of them."
        },
        "managed_device_count": {
          "$ref": "#/$defs/count",
          "description": "Devices in scope that the provider manages"
        },
        "mdm_enrolled_device_count": {
          "$ref": "#/$defs/count",
          "description": "Devices in scope with affirmative evidence of MDM enrollment"
        },
        "mdm_enrolled_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices in scope with affirmative evidence of MDM enrollment. Null when there are no devices"
        },
        "devices_by_os": {
          "type": "object",
          "propertyNames": {
            "$ref": "#/$defs/os_family"
          },
          "additionalProperties": {
            "$ref": "#/$defs/count"
          },
          "description": "Devices in scope by operating system family"
        }
      },
      "additionalProperties": false
    },
    "compliance": {
      "type": "object",
      "properties": {
        "compliant_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where the provider's or a baseline's compliance verdict passes, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"compliance.compliant_pct\"] counts the devices by outcome"
        },
        "basis": {
          "description": "Where the compliance verdict comes from: provider_reported when the provider decides it, or baseline_evaluation when the collector checks each device against the named baseline.",
          "enum": ["provider_reported", "baseline_evaluation"]
        },
        "baseline_id": {
          "description": "The baseline the collector checks devices against when the basis is baseline_evaluation.",
          "type": "string"
        },
        "baseline_version": {
          "description": "The version of that compliance baseline.",
          "type": "string"
        }
      },
      "dependentRequired": {
        "compliant_pct": ["basis"]
      },
      "additionalProperties": false
    },
    "hardening": {
      "type": "object",
      "properties": {
        "disk_encryption_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where every volume is encrypted at rest, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"hardening.disk_encryption_pct\"] counts the devices by outcome"
        },
        "screen_lock_compliant_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where the screen lock setting meets policy, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"hardening.screen_lock_compliant_pct\"] counts the devices by outcome"
        },
        "firewall_enabled_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where the host firewall is on, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"hardening.firewall_enabled_pct\"] counts the devices by outcome"
        }
      },
      "additionalProperties": false
    },
    "patching": {
      "type": "object",
      "properties": {
        "os_supported_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where the operating system version still gets security updates, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"patching.os_supported_pct\"] counts the devices by outcome"
        },
        "os_baseline_compliant_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where the operating system meets the named baseline, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"patching.os_baseline_compliant_pct\"] counts the devices by outcome"
        },
        "unsupported_os_count": {
          "$ref": "#/$defs/count",
          "description": "Devices whose operating system version no longer gets security updates"
        },
        "baseline_id": {
          "description": "The named OS baseline that os_baseline_compliant_pct checks devices against.",
          "type": "string"
        },
        "baseline_version": {
          "description": "The version of that OS baseline.",
          "type": "string"
        }
      },
      "dependentRequired": {
        "os_baseline_compliant_pct": ["baseline_id"]
      },
      "additionalProperties": false
    },
    "protection": {
      "type": "object",
      "properties": {
        "antimalware_installed_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where anti-malware is installed, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"protection.antimalware_installed_pct\"] counts the devices by outcome"
        },
        "antimalware_healthy_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where anti-malware is installed, running, and current, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"protection.antimalware_healthy_pct\"] counts the devices by outcome"
        },
        "edr_installed_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where an EDR agent is installed, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"protection.edr_installed_pct\"] counts the devices by outcome"
        },
        "edr_healthy_pct": {
          "$ref": "#/$defs/pct",
          "description": "Percentage of devices where an EDR agent is installed, running, and reporting, out of the devices whose reading passed or failed. Devices that are unknown or not applicable are left out. At the audit level, metric_coverage[\"protection.edr_healthy_pct\"] counts the devices by outcome"
        }
      },
      "additionalProperties": false
    },
    "metric_coverage": {
      "type": "object",
      "propertyNames": {
        "$ref": "#/$defs/metric_path"
      },
      "additionalProperties": {
        "$ref": "#/$defs/coverage"
      },
      "description": "Counts of devices for each percentage, by whether the device passed, failed, could not be read, or does not apply"
    },
    "availability": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["path", "reason"],
        "properties": {
          "path": {
            "description": "The metric this entry explains, named as in metric_coverage, such as hardening.screen_lock_compliant_pct.",
            "type": "string",
            "minLength": 1
          },
          "reason": {
            "enum": ["unsupported", "not_read", "permission_denied", "not_configured", "stale", "partial", "source_error"],
            "description": "unsupported when the vendor never reports the metric, and not_read when the vendor reports it but the collector does not read it"
          },
          "message": {
            "description": "A sentence that explains the reason, such as which vendor API reports the metric.",
            "type": "string",
            "maxLength": 200
          }
        },
        "additionalProperties": false
      },
      "description": "Why a metric has no reading, by path"
    },
    "devices": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/device"
      }
    }
  },
  "allOf": [
    {
      "if": {
        "properties": {
          "scope": {
            "properties": {
              "discovery_complete": {
                "const": false
              }
            }
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "maxProperties": 0
          }
        },
        "not": {
          "anyOf": [
            {
              "required": ["fleet"]
            },
            {
              "required": ["hardening"]
            },
            {
              "required": ["protection"]
            },
            {
              "required": ["patching"]
            },
            {
              "required": ["compliance"]
            }
          ]
        }
      }
    },
    {
      "if": {
        "required": ["compliance", "metric_coverage"],
        "properties": {
          "compliance": {
            "required": ["compliant_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["compliance.compliant_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["hardening", "metric_coverage"],
        "properties": {
          "hardening": {
            "required": ["disk_encryption_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["hardening.disk_encryption_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["hardening", "metric_coverage"],
        "properties": {
          "hardening": {
            "required": ["screen_lock_compliant_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["hardening.screen_lock_compliant_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["hardening", "metric_coverage"],
        "properties": {
          "hardening": {
            "required": ["firewall_enabled_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["hardening.firewall_enabled_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["patching", "metric_coverage"],
        "properties": {
          "patching": {
            "required": ["os_supported_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["patching.os_supported_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["patching", "metric_coverage"],
        "properties": {
          "patching": {
            "required": ["os_baseline_compliant_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["patching.os_baseline_compliant_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["protection", "metric_coverage"],
        "properties": {
          "protection": {
            "required": ["antimalware_installed_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["protection.antimalware_installed_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["protection", "metric_coverage"],
        "properties": {
          "protection": {
            "required": ["antimalware_healthy_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["protection.antimalware_healthy_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["protection", "metric_coverage"],
        "properties": {
          "protection": {
            "required": ["edr_installed_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["protection.edr_installed_pct"]
          }
        }
      }
    },
    {
      "if": {
        "required": ["protection", "metric_coverage"],
        "properties": {
          "protection": {
            "required": ["edr_healthy_pct"]
          }
        }
      },
      "then": {
        "properties": {
          "metric_coverage": {
            "required": ["protection.edr_healthy_pct"]
          }
        }
      }
    }
  ],
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
    "os_family": {
      "enum": ["windows", "macos", "linux", "ios", "ipados", "android", "chromeos", "other", "unknown"]
    },
    "metric_path": {
      "enum": ["compliance.compliant_pct", "hardening.disk_encryption_pct", "hardening.screen_lock_compliant_pct", "hardening.firewall_enabled_pct", "patching.os_supported_pct", "patching.os_baseline_compliant_pct", "protection.antimalware_installed_pct", "protection.antimalware_healthy_pct", "protection.edr_installed_pct", "protection.edr_healthy_pct"]
    },
    "coverage": {
      "type": "object",
      "required": ["passing_device_count", "failing_device_count", "unknown_device_count", "not_applicable_device_count"],
      "properties": {
        "passing_device_count": {
          "$ref": "#/$defs/count"
        },
        "failing_device_count": {
          "$ref": "#/$defs/count"
        },
        "unknown_device_count": {
          "$ref": "#/$defs/count"
        },
        "not_applicable_device_count": {
          "$ref": "#/$defs/count"
        }
      },
      "additionalProperties": false
    },
    "control_key": {
      "enum": ["disk_encryption", "screen_lock", "firewall", "os_supported", "os_baseline", "compliance", "antimalware_installed", "antimalware_healthy", "edr_installed", "edr_healthy"]
    },
    "control": {
      "type": "object",
      "required": ["state"],
      "properties": {
        "state": {
          "description": "The device's result for this control: pass, fail, unknown when there is no usable reading, or not_applicable when the control does not apply to the device.",
          "enum": ["pass", "fail", "unknown", "not_applicable"]
        },
        "observed_at": {
          "description": "When the reading behind this result was taken.",
          "$ref": "#/$defs/timestamp"
        }
      },
      "additionalProperties": false
    },
    "device": {
      "type": "object",
      "required": ["device_id", "os_family", "in_scope"],
      "properties": {
        "device_id": {
          "description": "The provider's identifier for the device.",
          "type": "string",
          "minLength": 1
        },
        "hostname": {
          "description": "The device's hostname, as the provider reports it.",
          "type": "string"
        },
        "os_family": {
          "description": "The device's operating system family.",
          "$ref": "#/$defs/os_family"
        },
        "os_name": {
          "description": "The operating system's name, as the provider reports it.",
          "type": "string"
        },
        "os_version": {
          "description": "The operating system's version, as the provider reports it.",
          "type": "string"
        },
        "assigned_user": {
          "type": "object",
          "required": ["user_id"],
          "properties": {
            "user_id": {
              "description": "The provider's identifier for the device's primary user.",
              "type": "string"
            },
            "login": {
              "description": "The primary user's sign-in name.",
              "type": "string"
            }
          },
          "additionalProperties": false
        },
        "management_state": {
          "description": "Whether the provider manages the device: managed, unmanaged, retired, or unknown when the provider does not say.",
          "enum": ["managed", "unmanaged", "retired", "unknown"]
        },
        "in_scope": {
          "type": "boolean"
        },
        "created_at": {
          "description": "When the provider first recorded the device.",
          "$ref": "#/$defs/timestamp"
        },
        "last_seen_at": {
          "description": "When the device last reported to the provider.",
          "$ref": "#/$defs/timestamp"
        },
        "controls": {
          "type": "object",
          "propertyNames": {
            "$ref": "#/$defs/control_key"
          },
          "additionalProperties": {
            "$ref": "#/$defs/control"
          }
        },
        "local_admin_accounts": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "account_id": {
                "description": "The provider's identifier for the local account.",
                "type": "string"
              },
              "username": {
                "description": "The local account's username on the device.",
                "type": "string"
              }
            },
            "additionalProperties": false
          }
        },
        "findings": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["control", "status"],
            "properties": {
              "finding_id": {
                "description": "The provider's identifier for this finding.",
                "type": "string"
              },
              "control": {
                "description": "The control the finding is about.",
                "$ref": "#/$defs/control_key"
              },
              "status": {
                "description": "Whether the issue is still open or has been resolved, or unknown when the provider does not say.",
                "enum": ["open", "resolved", "unknown"]
              },
              "first_seen_at": {
                "description": "When the issue was first recorded.",
                "$ref": "#/$defs/timestamp"
              },
              "last_seen_at": {
                "description": "When the issue was last seen.",
                "$ref": "#/$defs/timestamp"
              },
              "resolved_at": {
                "description": "When the issue was resolved. Absent while it is open.",
                "$ref": "#/$defs/timestamp"
              },
              "reason_code": {
                "description": "The provider's code for why the control failed on this device.",
                "type": "string"
              }
            },
            "additionalProperties": false
          }
        },
        "protection_modules": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["kind"],
            "properties": {
              "kind": {
                "description": "The kind of protection the module provides.",
                "enum": ["antimalware", "edr", "firewall", "device_control"]
              },
              "installed": {
                "type": "boolean"
              },
              "enabled": {
                "type": "boolean"
              },
              "healthy": {
                "type": "boolean"
              },
              "observed_at": {
                "description": "When the module's state was last reported.",
                "$ref": "#/$defs/timestamp"
              }
            },
            "additionalProperties": false
          }
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
  "provider": "jumpcloud",
  "organization_id": "acme-corp",
  "scope": {
    "discovery_complete": true,
    "observed_device_count": 120,
    "description": "provider_visible_devices",
    "max_observation_age_seconds": 604800
  },
  "fleet": {
    "devices_present": true,
    "device_count": 120,
    "managed_device_count": 120,
    "mdm_enrolled_device_count": 96,
    "mdm_enrolled_pct": 80,
    "devices_by_os": {
      "macos": 80,
      "windows": 36,
      "linux": 4
    }
  },
  "hardening": {
    "disk_encryption_pct": 94,
    "firewall_enabled_pct": 91
  },
  "protection": {
    "antimalware_installed_pct": 97,
    "antimalware_healthy_pct": 94
  },
  "metric_coverage": {
    "hardening.disk_encryption_pct": {
      "passing_device_count": 110,
      "failing_device_count": 6,
      "unknown_device_count": 4,
      "not_applicable_device_count": 0
    },
    "hardening.firewall_enabled_pct": {
      "passing_device_count": 101,
      "failing_device_count": 9,
      "unknown_device_count": 10,
      "not_applicable_device_count": 0
    },
    "protection.antimalware_installed_pct": {
      "passing_device_count": 34,
      "failing_device_count": 1,
      "unknown_device_count": 85,
      "not_applicable_device_count": 0
    },
    "protection.antimalware_healthy_pct": {
      "passing_device_count": 32,
      "failing_device_count": 2,
      "unknown_device_count": 86,
      "not_applicable_device_count": 0
    }
  },
  "availability": [
    { "path": "compliance.compliant_pct", "reason": "not_configured" },
    { "path": "hardening.screen_lock_compliant_pct", "reason": "unsupported" },
    { "path": "patching.os_supported_pct", "reason": "not_configured" },
    { "path": "patching.os_baseline_compliant_pct", "reason": "not_configured" },
    { "path": "protection.edr_installed_pct", "reason": "unsupported" },
    { "path": "protection.edr_healthy_pct", "reason": "unsupported" }
  ]
}
```

## 4. Field Mapping by Provider

| Normalized Field | JumpCloud |
|-----------------|-----------|
| `fleet.device_count` | Systems JumpCloud lists |
| `fleet.managed_device_count` | Every listed system, since each one runs the JumpCloud agent or is enrolled in MDM |
| `fleet.mdm_enrolled_device_count` | Systems enrolled in JumpCloud MDM |
| `hardening.disk_encryption_pct` | BitLocker on every volume on Windows. The agent's `fde.active` flag on other systems |
| `hardening.firewall_enabled_pct` | Application firewall state on macOS. Windows Security Center firewall health on Windows |
| `hardening.screen_lock_compliant_pct` | Not collected |
| `patching.os_supported_pct`, `patching.os_baseline_compliant_pct` | Not collected |
| `protection.antimalware_installed_pct` | An antivirus product registered with Windows Security Center |
| `protection.antimalware_healthy_pct` | An antivirus product that is on and has current signatures |
| `protection.edr_installed_pct`, `protection.edr_healthy_pct` | Not collected |
| `compliance.compliant_pct` | Not collected |

JumpCloud reports firewall and antimalware readings on Linux and mobile devices as unknown. It lists each metric it does not collect in `availability`.

## 5. Compliance Mapping

This schema supports the following compliance control families:

| Control | Field(s) Used | Condition Example |
|---------|--------------|-------------------|
| Device management | `fleet.mdm_enrolled_pct` | `>= 95` |
| Device encryption | `hardening.disk_encryption_pct` | `>= 95` |
| Screen lock | `hardening.screen_lock_compliant_pct` | `>= 95` |
| Host firewall | `hardening.firewall_enabled_pct` | `>= 95` |
| Malware protection | `protection.antimalware_installed_pct`, `protection.antimalware_healthy_pct` | `>= 95`, `>= 90` |
| Endpoint detection and response | `protection.edr_installed_pct`, `protection.edr_healthy_pct` | `>= 95`, `>= 90` |
| Supported operating systems | `patching.os_supported_pct`, `patching.unsupported_os_count` | `== 100`, `== 0` |
| Device compliance | `compliance.compliant_pct` | `>= 95` |

`patching.unsupported_os_count` is written at the audit and internal levels only.
