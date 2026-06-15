# `enabled_regions_otd_calculated_by_delivery_by` — Configuration Guide

Consul flag that decouples the new OTD calculation logic from the existing `delivery_by` feature flag. Only affects the `driver-app-api` service. Prerequisite ticket: [MOB-2859](https://gojitsu.atlassian.net/browse/MOB-2859).

> **Parent function:** [delivery-by.md](./delivery-by.md) · **Shared context:** [../../README.md](../../README.md)

---

## 1. Overview

| Attribute | Value |
|---|---|
| Feature / Config | New OTD calculation by region (independent of delivery_by) |
| Affected services | `driver-app-api` |
| Related tickets | [MOB-2859](https://gojitsu.atlassian.net/browse/MOB-2859) |

---

## 2. Configuration Levels & Precedence

This config has a single **Global** level — one Consul key whose value contains the list of enabled regions.

| Level | Applies to | Storage / scope | Default |
|---|---|---|---|
| Global | Entire system | Consul | `""` (empty string — feature off) |

> No separate Region / Warehouse / Driver level. The target regions are embedded directly in the key **value** (e.g. `SAT,SFO`).

---

## 3. Configuration Reference

### 3.1 `apps.driverappapi.delivery.enabled_regions_otd_calculated_by_delivery_by` — New key

| Field | Value |
|---|---|
| Type | String (comma-separated region codes) |
| Storage | Consul |
| Used by | `driver-app-api` → `AssignmentTrafficManagerV2` |
| Purpose | Enables the new OTD calculation per region, independently of the `delivery_by` flag. When a region is in the list, the `specialtyClientIds` filter is bypassed — all shipments in that region use the new OTD logic. |
| Default | `""` (empty string — feature fully off) |
| Allowed values | Uppercase region codes separated by commas (e.g. `SAT,SFO,LAX`); or `*` to enable all regions |
| Configurable at | Global |
| Required before | Enabling new OTD calculation for a specific region without affecting the `delivery_by` flag |

**Example values:**

```text
# Disabled (default)
(key absent or empty)

# Enable for SAT only
SAT

# Enable for multiple regions
SAT,SFO,LAX

# Enable for all regions
*
```

**Consul paths:**

```text
Staging:    staging/apps/driverappapi/delivery/enabled_regions_otd_calculated_by_delivery_by
Production: apps/driverappapi/delivery/enabled_regions_otd_calculated_by_delivery_by
```

---

## 4. Behavior per Value

| Value | Behavior |
|---|---|
| `""` / key absent | `specialtyClientIds` filter works as normal — only specialty clients use new OTD |
| `SAT` | SAT assignments: filter bypassed, all clients use new OTD calculation |
| `SAT,SFO` | Same, applies to both SAT and SFO |
| `*` | Filter bypassed for all regions |

> **Note:** This flag only controls the `specialtyClientIds` filter step inside `AssignmentTrafficManagerV2`.

---

## 5. Apply / Restart Checklist

| Service | Type | Restart required? | Reason | Order |
|---|---|---|---|---|
| `driver-app-api` | API | **Yes** | Consul key is read in the `AssignmentTrafficManagerV2` constructor at startup — not hot-reloaded | 1 |

