---
# ===== IDENTITY =====
feature: separated-bonus-from-route-base-pay
title: Separated Bonus from Route Base Pay
domain: driver-app
sub_domain: payout
schema_version: 1.0

# ===== STATE =====
status: active
maturity: evolving
maintainer:

# ===== SEARCH & DISCOVERY =====
keywords:
  - bonus pay
  - route base pay
  - separate_bonus_pay_and_route_pay
  - feature flag
  - TXCreator
  - driver-app-api
  - IC driver
  - payout transaction
  - OUTVOICE_BONUS

# ===== TAXONOMY =====
user_types:
  - IC driver
  - Ops (per-region flag enablement)
system_touchpoints:
  - TXCreator (payout pipeline) — BE
  - AppFeaturesManager — BE
  - event-processors (worker) — BE
  - driver-app-api (BonusUtils, decorators, Mongo HTML templates) — BE
  - Driver App mobile client (bonus chip, tooltip breakdown) — FE

# ===== EXTERNAL SOURCES =====
confluence_refs:
  - id: 2465169409
    title: Separate Bonus Pay from Route Base Pay
    url: https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2465169409/Separate+Bonus+Pay+from+Route+Base+Pay

figma_refs:
  - name: Separated Bonus from Route Base Pay
    url: https://www.figma.com/design/oko9PhlOmF6xNqhRioySxy/Driver-App?node-id=22664-78662&t=Hrb7P0TRNn3ltWoa-1

# Separated Bonus from Route Base Pay

> **Scope** — How bonus pay is split from route base pay in the payout pipeline and driver-app UI when `separate_bonus_pay_and_route_pay` is enabled per region. Covers IC drivers only; DSP/3P payout and display are out of scope (unchanged).

---

## 1. Actors & Systems

- **IC driver** — sees separated bonus/base pay in the driver app
- **Ops** — enables the feature flag per region
- **`TXCreator`** — payout pipeline component that creates payout transactions per assignment
  - Key fields: `TYPE_ROUTE_DRIVING`, `OUTVOICE_BONUS`
- **`AppFeaturesManager`** — resolves the `separate_bonus_pay_and_route_pay` flag per region
  - Key fields: `AppFeature.Features.SEPARATE_BONUS_PAY_FEATURE`
- **`BonusUtils`** (driver-app-api) — splits `tourCost` into `routeBasePay + routeBonusPay` for display

---

## 2. Business Rules

### 2.1. Feature Flag

| # | Attribute | Value |
|---|---|---|
| 1 | Name | `separate_bonus_pay_and_route_pay` |
| 2 | Scope | Region (`APP_FEATURES`, `AP_DEFAULT`) |
| 3 | Default | `false` — must be explicitly enabled per region via AppFeaturesManager |
| 4 | Constant | `AppFeature.Features.SEPARATE_BONUS_PAY_FEATURE` (dispatch-bizlogic) |

### 2.2. Transaction Rules (flag ON, IC only)

**Allow when ANY condition is true:**

| # | Condition |
|---|---|
| A1 | `bonus > 0` → separate `OUTVOICE_BONUS` transaction(s) created alongside `TYPE_ROUTE_DRIVING` |

**Block when ANY condition is true:**

| # | Condition |
|---|---|
| B1 | `bonus == 0` → only `TYPE_ROUTE_DRIVING` created, no bonus rows |
| B2 | `bonus < 0` → deduction folded into base pay, no separate bonus transaction |
| B3 | Driver is DSP/3P → transaction unchanged regardless of flag state |

| Transaction | Type | Note |
|---|---|---|
| Route base pay | `TYPE_ROUTE_DRIVING` | — |
| Route bonus | `OUTVOICE_BONUS` | `[label] Route Bonus` |
| Driver bonus | `OUTVOICE_BONUS` | `[label] Driver Bonus` |

**Core rule**: Flag ON + IC + bonus > 0 → 3 transactions instead of 1; everything else (flag OFF, DSP/3P, zero/negative bonus) stays a single combined transaction.

### 2.3. UI Rules (driver-app-api)

**Allow when ANY condition is true:**

| # | Condition |
|---|---|
| A1 | Flag ON → `routeBasePay` shown as primary amount, "Bonus: +$X.XX" chip visible for route bonus |

**Block when ANY condition is true:**

| # | Condition |
|---|---|
| B1 | Flag OFF → total pay shown as before, no "Bonus" chip |
| B2 | DSP/courier assignment → no bonus separation, flag ignored |

- `BonusUtils.separateBonus()` checks flag and splits `tourCost` into `routeBasePay + routeBonusPay`
- Decorators affected: `ActiveAssignmentDecorator`, `ReservableAssignmentDecorator`, `OpenAssignmentDecorator`
- `LayoutGroup.Subsidy` shows "Bonus: +$X.XX" chip for route bonus when flag ON
- Mongo HTML templates for acceptance screen must have `route_bonus_area` and `driver_bonus_area` slots — excludes DSP templates (manual post-deploy step)

### 2.4. Special Cases

- Driver bonus chip label updated: `+$X bonus` → `Driver bonus: +$X` — immediate, applies regardless of flag
- Fuel subsidy no longer bundled into displayed price — immediate, applies regardless of flag
- `tourCost` and `bonus` on `Assignment` are nullable — all calculation paths guard for null

---

## 3. Workflow

1. IC driver completes assignment with bonus
2. `TXCreator.processAssignment` calculates total pay (route pay + bonuses)
3. Check `separateBonusPay` flag for the driver's region
4. If **TRUE**:
   - Create transaction `TYPE_ROUTE_DRIVING` (base route pay)
   - Create transaction `OUTVOICE_BONUS` (route bonus, if `bonus > 0`)
   - Create transaction `OUTVOICE_BONUS` (driver bonus, if IC and `bonus > 0`)
   - Tracking event emits `driver_bonus` fact: `receivedAmount − tourCost − routeBonus`
5. If **FALSE**:
   - Create single transaction `TYPE_ROUTE_DRIVING` (total pay)
6. Driver app displays chips/tooltip breakdown per flag state (section 2.3)

---

## 4. Data Model

### Tables

- **`item_metadata`** — stores the feature flag
  - `scope`: `"APP_FEATURES"`
  - `owner`: `"AP_DEFAULT"`
  - `values`: includes `separate_bonus_pay_and_route_pay` when enabled for a region

---

## 5. Related Behaviors

- **Payout pipeline restart**: `event-processors` (worker) requires a restart — new `AppFeaturesManager` + `MetadataDAO` wired at startup, `clientCache` removed.
- **Driver app restart**: `driver-app-api` requires a restart — new `BonusUtils` wired at startup.
- **DSP/3P payout**: unaffected — see DSP payout flow (not covered in this doc).

---

## 6. Known Issues, Gotchas & Ambiguities

### Currently open issues

None currently.

### Gotchas

- **Mongo template rollout**: acceptance screen HTML templates for non-DSP flows must be manually updated post-deploy to include `route_bonus_area` and `driver_bonus_area` slots — not automated.
- **Null-safety**: `tourCost` and `bonus` are nullable on `Assignment`; missing guards would break transaction creation.