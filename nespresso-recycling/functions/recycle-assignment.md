---
feature: recycle-assignment
title: Recycle Assignment (Completion)
domain: nespresso-recycling
sub_domain: recycle-assignment
schema_version: 1.0
status: active
maturity: evolving
maintainer: hue.chu
keywords:
  - recycle assignment
  - complete recycle assignment
  - recycle pickup return
  - PICKUP_SCAN_COMPLETED
  - recycle stop
  - Nespresso recycling
user_types:
  - Driver
system_touchpoints:
  - driver-app-api
  - recycle_assignments
  - recycle_stops
  - recycle_packages
  - shipment_outbound_events
---

# Recycle Assignment (Completion)

> **Scope** — Completing a recycle pickup-return assignment via driver-app-api: the pickup-side gate
> that decides whether the assignment can be completed. Out of scope: the scan / Done actions
> (event-driven, another service) and the deposit/return internals.

---

## 1. Actors & Systems

- **Driver** — performs the recycle pickup-return and completes the assignment.
- **driver-app-api** — `completeRecycleAssignment` + `POST /recycle-package/recycle-assignment/complete`.
- **`recycle_assignments`** — the assignment. Key fields: `status`, `ref_driver_id`, `warehouse_id`.
- **`recycle_stops`** — pickup/return stops. Key fields: `type` (PICKUP/RETURN), `status`.
- **`recycle_packages`** — Key fields: `scanned_package_codes`, `pickup_ts`.
- **`shipment_outbound_events`** — where the `GATHER_COMPLETED` event is written.

---

## 2. Business Rules

### 2.1. Completing the assignment (pickup gate)

**Allow (completion succeeds) when ALL pickup stops are finished:**

| # | Condition |
|---|---|
| A1 | Every PICKUP stop `status` ∈ {`PICKUP_COMPLETED`, `PICKUP_FAILED`, `PICKUP_SCAN_COMPLETED`} |

**Block (cannot complete) when ANY condition is true:**

| # | Condition |
|---|---|
| B1 | Any PICKUP stop `status` = `PICKUP_PENDING`, `PICKUP_READY`, or `null` → 412 "There are some unfinished pickup recycling stops"; assignment stays `IN_PROGRESS` |

**Core rule**: a scanned-but-not-confirmed pickup stop is good enough to complete; a never-attempted one is not.

### 2.2. Special Cases

- The return-side gate is **not enforced** by this endpoint (commented out in code); RETURN status does not affect completion here.

---

## 3. Workflow

1. Driver scans each recycled package → pickup stop = `PICKUP_SCAN_COMPLETED`.
2. Driver taps Done → pickup stop = `PICKUP_COMPLETED` (normal flow; emits `GATHER_COMPLETED`).
3. If Done is lost (crash / network / left early) → pickup stop stays `PICKUP_SCAN_COMPLETED`.
4. Driver completes the assignment → the pickup gate (§2.1) decides: succeed or 412.

---

## 4. Data Model

### Tables (MongoDB `prod`)

- **`recycle_assignments`** — `status`: IN_PROGRESS → COMPLETED; `is_active`; `warehouse_id`; `ref_assignment_ids[]`; `ref_driver_id`.
- **`recycle_stops`** — `type`: PICKUP/RETURN; `status`; `recycle_package_id`; `recycle_assignment_id`; `ref_stop_id`; `ref_shipment_id`.
- **`recycle_packages`** — `scanned_package_codes[]`; `pickup_ts`; `returned_ts`; `recycle_assignment_id`.
- **`shipment_outbound_events`** — the `GATHER_COMPLETED` event log.

### Reference query

```text
// Mongo (read-only) — inspect a recycle assignment's stops
db.recycle_stops.find({ recycle_assignment_id: "<id>" }, { type: 1, status: 1 })
```

---

## 5. Related Behaviors

- **Tracking / analytics**: completing while a stop stays `PICKUP_SCAN_COMPLETED` does not emit
  `GATHER_COMPLETED` for that stop — a tracking gap only (no pay / shipment-status impact;
  `pickup_ts` is still stamped). See [recycling-feature-overview](recycling-feature-overview.md).
- **Stop-status source**: `PICKUP_SCAN_COMPLETED` / `PICKUP_COMPLETED` are set by an inbound event
  flow (`update_recycle`), not by the completion endpoint.

---

## 6. Known Issues, Gotchas

### Gotchas

- `PICKUP_SCAN_COMPLETED` always has `scanned_package_codes` (scanning always yields a QR code) — the empty-codes case does not occur.
- The return-side gate is commented out → RETURN status does not affect completion on this endpoint.

### Fixed issues (kept for regression coverage)

- **[FIXED — 2026]** A recycle assignment was blocked when a pickup stop was stuck at `PICKUP_SCAN_COMPLETED`; it is now accepted as finished so completion succeeds.

---

## 7. References

### Confluence

- [Pickup Return/Recycle Packages E2E](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1594621999/Pickup+Return+Recycle+Packages+E2E)
- [Enabling Recycling Feature for Drivers: Configuration and Setup Guide](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1915551790/Enabling+Recycling+Feature+for+Drivers+Configuration+and+Setup+Guide)
