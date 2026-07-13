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

---

# `specialty_client_ids` — Configuration Guide

List of client IDs treated as "specialty". Shipments belonging to these clients are **always excluded**
from Driver OTD measured by the dynamic `delivery_by_ts` (MOB-2934) and are instead measured against the
original SLA (`dropoff_latest_ts` / internal-OTD 8 PM) — **even when the region already has
measure-by-delivery_by enabled**.

> **Parent function:** [delivery-by.md](./delivery-by.md) · **Related:** [`enabled_regions_otd_calculated_by_delivery_by`](#enabled_regions_otd_calculated_by_delivery_by--configuration-guide) (above)

## 1. Overview

| Attribute | Value |
|---|---|
| Config key | `specialty_client_ids` |
| Owner | ⚠️ TBD — confirm with team (Mobile / Dispatch) |
| Affected services | `driver-app-api` (reads directly); indirectly affects the Driver OTD calculation engine |
| Related tickets | MOB-2862 (exclude specialty/SAME_DAY/FCTD from dynamic deliver-by), MOB-2934 |
| Related configs | `enabled_regions_otd_calculated_by_delivery_by`, `delivery_window_by_region` (field `midnight_local`) |

## 2. Configuration Levels & Precedence

A single **Global** key holding a comma-separated list of client IDs. There is no region/warehouse/driver
override — "which clients it applies to" is expressed by the value itself (membership by `shipment.client_id`).

| Level | Applies to | Storage / scope | Default |
|---|---|---|---|
| Global | Entire system; lists clients treated as specialty | ⚠️ TBD — Consul KV key `specialty_client_ids` (full path unconfirmed) | Empty (`""`) → no client is specialty |
| Region / Warehouse / Driver | ❌ Not supported | — | — |

**Resolution:** a shipment is "specialty" ⟺ `shipment.client_id` ∈ this list. This condition is
independent of and **additive** with `enabled_regions_otd_calculated_by_delivery_by` (see Section 4).

## 3. Configuration Reference

### 3.1 `specialty_client_ids`

| Field | Value |
|---|---|
| Type | String — comma-separated (numeric) client IDs (parsed → `List<Long>`; trimmed, blanks dropped, `Long.parseLong`) |
| Storage | Consul, read via `config.getStringAttribute(...)` in `driver-app-api` ⚠️ (full Consul path: TBD) |
| Used by | `driver-app-api` → `AssignmentTrafficManagerV2` (constructor, ~lines 213–221) |
| Purpose | Marks shipments of these clients as "specialty" → excluded from OTD-by-delivery_by; measured against the original SLA |
| Default | Empty → empty list → no client marked specialty (only the region condition applies) |
| Allowed values | Valid client IDs, integers (`Long`) |
| Configurable at | Global (one key, lists client IDs) |
| Required before | To exclude a client from dynamic OTD, that client ID must be present in the list |

**Example value:**

```text
1234,5678,9012
```

## 4. Behavior per Value / State

A shipment is added to `estimated_delivery_window.specialty_client_shipment_ids` (→ excluded from
dynamic OTD, measured by `dropoff_latest_ts` / 8 PM) when **either** condition holds:

```text
!isEnabledRegionsOtdCalculatedByDeliveryBy(assignment)   // (A) region not enabled
|| specialtyClientIds.contains(s.clientId)               // (B) client is in specialty_client_ids
```

| Case | `shipment.client_id` ∈ `specialty_client_ids`? | Region enabled? | OTD measured by |
|---|---|---|---|
| 1 | No | Yes | dynamic `delivery_by_ts` (MOB-2934) |
| 2 | Yes | Yes | `dropoff_latest_ts` / 8 PM (excluded) |
| 3 | No | No | `dropoff_latest_ts` / 8 PM (whole route excluded) |
| 4 | Yes | No | `dropoff_latest_ts` / 8 PM |

→ `specialty_client_ids` is a **per-client exemption list**: it always wins to exclude a shipment from
dynamic OTD, regardless of region.

## 5. Apply / Restart Checklist

| Service | Type | Restart required? | Reason | Order |
|---|---|---|---|---|
| `driver-app-api` | API | **Yes** | Config is read once at startup (`AssignmentTrafficManagerV2` constructor); no Consul hot-reload | 1 |

## 6. Validation & Troubleshooting

- **Verify it loaded:** startup log — `"Specialty client ids: [...]"` (`AssignmentTrafficManagerV2` ~line 232).
- **Symptom:** a client's shipment is still scored by `delivery_by` despite wanting it excluded → check the `client_id` is actually in the list and the value is numeric (parsed as `Long`; stray characters throw / cause it to be skipped).
- After editing Consul → **restart** `driver-app-api`.

## 7. Related References

- MOB-2862 — Exclude specialty/SAME_DAY/FCTD from dynamic deliver-by
- MOB-2934 — Allow deliver by to exceed 8 PM cutoff
- Related configs: `enabled_regions_otd_calculated_by_delivery_by`, `delivery_window_by_region.midnight_local`
- Owner/contact: ⚠️ TBD

