# Linehaul Driver App — Project Documentation

> **Project:** Linehaul Driver App MVP
> **Target launch:** Early Q3 2026
> **Source-of-truth specs:** Confluence (Engineering space) → Driver App Logic
> **Last updated:** June 2026

Web app for the **Driver Linehaul** domain. Per-function documentation lives in the
[`functions/`](functions/) directory.

## Function index

| Function | File | Status |
|----------|------|--------|
| Log-on (Login) | [functions/log-on.md](functions/log-on.md) | Draft |
| TPA (Third-Party Agreement) Signing Guard | [functions/contract.md](functions/contract.md) | Draft |

> To add a new function: copy [`functions/_template.md`](functions/_template.md) to
> `functions/<function-name>.md`, then add a row to the table above.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture — Systems Integration](#2-architecture--systems-integration)
3. [DSP Driver Lifecycle & States](#3-dsp-driver-lifecycle--states)
4. [Driver Account Creation Flows](#4-driver-account-creation-flows)
5. [Linehaul Trip Types](#5-linehaul-trip-types)
6. [Related Jira Tickets](#6-related-jira-tickets)
7. [References](#7-references)
8. [Open Questions](#8-open-questions)
9. [Contributors](#9-contributors)

> The **Log-on / Login Screen** function is documented separately → [functions/log-on.md](functions/log-on.md).

---

## 1. Project Overview

The **Linehaul Driver App** is a mobile application for drivers operating Linehaul shipments within the Jitsu logistics network. It supports two driver populations:

- **DSP Linehaul drivers** — employed by Linehaul DSP partners contracted with Jitsu
- **3P (third-party / spot) drivers** — independent carriers brought in for individual shipments

### MVP scope (Q3 2026)

The MVP focuses on **core route completion** functionality:

- Driver claims an assigned route
- Picks up loads at origin
- Submits telemetry during transit
- Docks at destination
- Records key events (event sourcing architecture)

Auto-provisioning of drivers and carrier creation are **deprioritized** for the initial MVP release.

### Out of scope (for MVP)

- Full auto-credentialing flow (will use existing `linehaul_checker` worker)
- IC (Independent Contractor) driver flow
- Reset password complexity rules (separate ticket)
- 2FA for login

---

## 2. Architecture — Systems Integration

The Linehaul flow involves three primary external systems plus the Jitsu backend.

### System responsibilities

| System | Role | What it owns |
|---|---|---|
| **NetSuite** | ERP — financial system of record | Load creation, carrier contracts, PO, invoicing, billing |
| **Jitsu Backend (mm-service)** | Operations bridge | Driver records, dispatching, route assignment, app data |
| **Project44** | Visibility & tracking platform | Real-time GPS tracking, ETAs, geofencing, exception alerts |
| **Project44 DriveView app** | Driver-facing tracking app | GPS tracking for drivers (separate from Jitsu Linehaul Driver App) |

### Data flow (Linehaul shipment lifecycle)

```
NetSuite (load created)
    ↓ API webhook
Jitsu Backend (mm-service)
    ↓ register trip
    ├─→ Auto-provision driver via SMS (if 3P)
    ├─→ Send driver phone to Project44 for tracking
    └─→ Surface assignment to Jitsu Linehaul Driver App

Driver (during shipment)
    ↓ uses Jitsu Linehaul Driver App (business workflow)
    ↓ may also use Project44 DriveView (GPS tracking)
        ↓
Project44 (tracking events)
    ↓ webhooks back
Jitsu Backend
    ↓ exception alerts, ETA updates
    ↓ feed into Jitsu dashboards

NetSuite (post-delivery)
    ↓ invoicing, settlement, billing
```

### Two GPS tracking options being considered

| Option | Description | Status |
|---|---|---|
| **A — Use DriveView for GPS** | Driver installs both apps; DriveView tracks GPS, Jitsu app handles workflow | Currently in use |
| **B — Jitsu app tracks GPS natively** | Single app for everything; Jitsu integrates GPS directly | Future consideration |

The VPI process mapping indicates **Option A** is current direction.

---

## 3. DSP Driver Lifecycle & States

### Canonical spec

📄 **Confluence (source of truth):** [DSP Driver — Lifecycle, States & Behavior Specification](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2481356801)

### State machine summary

Every DSP Driver record is defined by three fields: **Status**, **Background Status**, and **In DSP?**

| Status | Background Status | In DSP? | Meaning | App Access |
|---|---|---|---|---|
| `CREATED` | `PENDING` | Yes | Newly created, awaiting Jitsu manual approval | ✅ Can login |
| `BACKGROUND_APPROVED` | `MANUAL_APPROVED` | Yes | Approved and operational | ✅ Can login |
| `QUIT` | `MANUAL_APPROVED` | Yes | Driver quit (still in DSP) | ✅ Can login |
| `QUIT` | `MANUAL_APPROVED` | No | Deleted by admin (no DSP) | ❌ Force logout / cannot login |
| `SUSPENDED` | `MANUAL_APPROVED` | Yes | Suspended by dispatcher | ✅ Can login |
| `CREATED` | `NULL` | No (IC) | Re-activated after delete — converted to Independent Contractor | ✅ Can login (re-onboard) |

### Key transitions

| ID | Name | Effect |
|---|---|---|
| **T1** | Create Driver | Auto-provisioned from source-of-truth ERP |
| **T2** | Jitsu Approve | Background check completed |
| **T3** | Driver Quits | Self-initiated quit (still in DSP) |
| **T4** | Re-continue | Dispatcher reverses quit |
| **T5** | Suspend | Dispatcher suspends |
| **T6** | Re-activate | Dispatcher lifts suspension |
| **T7** | Admin Delete | Permanent removal — sets `courier_drivers.is_active = false`, `deleted = true` |
| **T8** | Re-activate after Delete | Converts driver to IC (no longer DSP) |

### Key principles

- **Only the deleted state blocks login.** All other states allow login (app-side gating restricts operational actions).
- **Login ≠ Active.** Authentication and authorization are separate.
- **Re-activation after delete converts driver to IC**, not DSP. From that point, IC Driver lifecycle applies (separate spec, TBD).

---

## 4. Driver Account Creation Flows

Two parallel flows exist for creating Linehaul drivers, both confirmed valid by Product Owner.

### Flow A — DSP Linehaul Driver

```
1. Jitsu creates a courier with type LINEHAUL
2. DSP company creates their drivers in their portal
3. Jitsu approves the driver
4. Driver logs in with DSP-provided credentials
```

> ⚠️ **Open question:** Whether DSP Linehaul drivers follow the standard DSP background check process (`CREATED + PENDING` → `BACKGROUND_APPROVED + MANUAL_APPROVED`) or are activated immediately. PO has given partially conflicting answers — pending clarification.

### Flow B — 3P (Third-Party) Driver Auto-Provisioning

```
Trigger: NetSuite load assigned to driver, within 7 days of pickup_date

If phone number NOT in system:
  1. Auto-create driver profile (placeholder if name missing)
  2. Send SMS with app download link + password reset link
  3. Driver opens app, resets password
  4. Driver logs in using phone number + new password

If phone number ALREADY in system:
  → Skip — no duplicate SMS sent
```

### Exception handling

- **Invalid phone number (landline / bounceback):** flag operational exception for manual resolution
- **Carrier name from NetSuite doesn't match Jitsu courier:** currently manual (auto-create option under consideration)

### Carrier-driver-vendor relationship

```
Driver  ─── tied to ───►  Vendor (carrier)  ─── tied to ───►  Region(s)
                                  │
                                  └──► TPA (standardized across regions)
```

- Drivers are tied to **vendors**, not directly to regions
- A vendor can span multiple regions
- TPAs (Third-Party Agreements) are standardized and do not vary by region
- Linehaul drivers use the existing **DSP TPA type** (no new TPA type)

### Operational note: warehouse license scan

Capturing a photo of the driver's license at the warehouse is **NOT** part of the account creation workflow. It exists only for:
- **Timestamping** — official arrival time for offloading duration tracking, prevents invalid detention fee claims
- **Anti-fraud** — verify driver identity (3P carriers frequently substitute drivers last-minute)

---

## 5. Linehaul Trip Types

The `trip_type` (in new DB `trips`) / `linehaul_type` (in old DB `linehauls`) field categorizes the operational nature of each shipment.

| Type | Description |
|---|---|
| **First Mile** | Shipper → Jitsu warehouse (initial leg) |
| **Middle Mile** | Warehouse → warehouse transfer (between Jitsu facilities) |
| **MH Middle Mile - Line Haul** | Long-haul warehouse-to-warehouse involving a Mobile Hub |
| **MH - Pallet Return** | Empty pallet return leg from a Mobile Hub |
| **MM - Pallet Return** | Empty pallet return leg from Middle Mile route |
| **Pallet Returns** | Generic empty pallet return |
| **Mobile Hub** | Operational moves of the Mobile Hub itself |
| **Hub Rent** | Cost-tracking entry (not actual transport) |
| **Brokerage** | Outsourced shipment to third-party broker |
| **Planning** | Trip in planning state (not executed yet) |

### Type definitions reference

- **Mobile Hub (MH)** — A temporary / mobile cross-dock or yard facility used in hub-and-spoke logistics. Flexible by season or region demand.
- **Middle Mile (MM)** — Inter-warehouse transfer leg, distinct from First Mile (shipper → warehouse) or Last Mile (warehouse → customer).
- **Line Haul** — Long-haul truck transportation, typically 53' trailers, often over significant distances.

---

## 6. Related Jira Tickets

### Epic

- **MOB-2004** — Linehaul Driver App (parent epic)

### Active / In-progress tickets

| Ticket | Title | Status | Assignee |
|---|---|---|---|
| MOB-2334 | Frontend OAuth | ✅ Done | — |
| MOB-2543 | Enriched auth spec | Reference | — |
| MOB-2550 | Backend DSP-type gating | 🟡 Blocked | Bach Mai |
| MOB-2606 | Login crash bug (non-existent gojitsu.com account) | 🔴 To Do | Bach Mai |
| MOB-2722 | Auto-Credentialing via NetSuite Integration | Investigating | — |

### Current app version

- **Linehaul Driver App v2.8.0**

---

## 7. References

### Confluence (canonical specs)

- [Driver App Logic (parent page)](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/530776065)
- [DSP Driver — Lifecycle, States & Behavior Specification](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2481356801)
- [Middle Mile Service — NetSuite Integration](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2500788234)
- [Knowledge base for Linehaul in mm-service](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2538963062)

### External system documentation

- **NetSuite** — Source-of-truth ERP for orders, carriers, billing
- **Project44** — Visibility platform: https://www.project44.com
- **Project44 DriveView** — Driver mobile app: https://www.project44.com/products/driveview

### Internal Jitsu tools

- **`linehaul_checker` worker** — Existing backend worker in mm-service responsible for syncing data from NetSuite
- **mm-service** — Middle Mile service repository

---

## 8. Open Questions

Items awaiting Product Owner / Engineering clarification:

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | Does DSP Linehaul follow normal DSP background check, or skip it? | PO | Pending |
| 2 | Should carrier auto-create when NetSuite `carrier_name` doesn't match existing Jitsu courier? | PO | Investigating |
| 3 | Login form: single field for username/email/phone or 3 separate fields? | PO | Pending |
| 4 | Token TTL — confirm 7 days? | Backend | Pending |
| 5 | Account lockout policy — N attempts, lockout duration? | PO + Cognito | Pending |
| 6 | What is the cutover plan from `linehauls` (old DB) to `trips` (new DB)? | Backend | Pending |

---

## 9. Contributors

- **Hue Chu** — Product Owner, Mobile Team (Linehaul Driver App)
- **Bach Mai** — Mobile engineering
- **Dustin Graham** — Engineering (event sourcing architecture)
- **Lina Wong** — Project coordination
- **Mi Nguyen** — Coordination with Vietnam team
- **Sampson Wu, Pete Foradori, Evan Jordan, Chantra Park** — Contributors to VPI process mapping
