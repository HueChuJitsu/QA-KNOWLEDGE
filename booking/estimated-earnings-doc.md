# Estimated Earnings & Typical Driver Earnings Range Documentation

## Related Tickets

| Ticket | Title | Status |
|--------|-------|--------|
| [MOB-2518](https://gojitsu.atlassian.net/browse/MOB-2518) | [BE] Calculate Estimated Earnings & Typical Driver Earnings Range | DONE |
| [MOB-2517](https://gojitsu.atlassian.net/browse/MOB-2517) | [BE] Display Estimated Earnings & Typical Driver Earnings Range | DONE |
| [MOB-2541](https://gojitsu.atlassian.net/browse/MOB-2541) | [BE] Store Estimated Earnings & Typical Driver Earnings Range | DONE |
| [MOB-2693](https://gojitsu.atlassian.net/browse/MOB-2693) | [BE] Display Estimated Earnings & Typical Driver Earnings Range in Booking session details screen | DONE |
| [MOB-2841](https://gojitsu.atlassian.net/browse/MOB-2841) | [BE] Refine driver earnings range: forecast pre-routing, actual Booking Session values after routing | STAGING REVIEW FINISHED |

---

## 1. Overview

To improve pay transparency for drivers, the Driver App displays three earnings fields alongside the absolute min/max pay range on booking ticket cards. Values are calculated from historical completed route data and refined once routing completes.

The three fields displayed are:

- **Estimated Earnings** — a single value (P50 pre-routing; routing zone average after routing)
- **Most Drivers Earn** — a range (P20–MAX by default; switches to actual routing range after routing)
- **Earnings Range** — absolute min/max (min always from pay formula; max from forecast pre-routing, routing max after routing)

The feature is controlled by a region config flag (`enabledRegions` in Consul) for gradual rollout.

> **MOB-2841 refinements:** Each field now has a **pre-routing (forecast)** value and an **after-routing** value, switching automatically once routing is confirmed. The absolute minimum is now sourced from the pay formula (WAT-2086) rather than historical data.

---

## 2. Configuration

| Environment | Config Path |
|-------------|-------------|
| Staging | `consul-staging → apps/driverappapi/estimated_earnings_cfg` |
| Production | `consul-prod → apps/driverappapi/estimated_earnings_cfg` |

Key config fields:

| Field | Description | Default |
|-------|-------------|---------|
| `enabledRegions` | Regions where the feature is active (e.g. `["JFK", "CLE", "LAX", "PHX"]`) | — |
| `minSampleSize` | Minimum zone-level route count before using ALL_ZONE fallback | 5 |
| `minTypicalEarningsSpreadUsd` | Minimum $ spread enforced on Most Drivers Earn when sample < minSampleSize and spread < threshold | 10.0 |
| `typicalEarningsMin` | Percentile for Most Drivers Earn lower bound | P20 |
| `typicalEarningsMax` | Percentile for Most Drivers Earn upper bound | MAX |
| `averageEarnings` | Percentile for Estimated Earnings (pre-routing) | P50 |

---

## 3. Ticket Breakdown

| Ticket | Title | Scope | Status |
|--------|-------|-------|--------|
| MOB-2518 | [BE] Calculate Estimated Earnings & Typical Driver Earnings Range | Worker calculates and stores earnings stats in `route_tour_cost_stats` MongoDB collection | DONE |
| MOB-2517 | [BE] Display Estimated Earnings & Typical Driver Earnings Range | BE API exposes calculated values; displayed on total booked ticket & active ticket screens | DONE |
| MOB-2541 | [BE] Store Estimated Earnings & Typical Driver Earnings Range | Stores snapshot of 3 earnings fields in ticket MongoDB collection at time of booking | DONE |
| MOB-2693 | [BE] Display Estimated Earnings & Typical Driver Earnings Range in Booking session details screen | BE API exposes values for booking session (BS) details screen | DONE |
| **MOB-2841** | **[BE] Refine driver earnings range: forecast pre-routing, actual Booking Session values after routing** | Refines sourcing logic: pre/post-routing distinction, pay-formula min (WAT-2086), spread floor, clamping invariant. BE-only; no FE changes. | STAGING REVIEW FINISHED |

---

## 4. Feature Logic & Architecture

### 4.1 Calculation (MOB-2518)

A background worker (`worker-est-route-time-aggregation`) runs on a schedule to calculate earnings statistics from completed assignment data.

- **Data source:** `assignments` table — completed routes only (`status = COMPLETED`, `courier_id IS NULL`, `tour_cost > 0`)
- **Lookback window:** trailing 21 days (configured via `tour_cost_trailing_window_in_days` in `mongo.schedule`; falls back to `trailing_window_in_days` if not set)
- **Grouping:** by zone + region + day of week
- **Fallback:** if zone-level count < `minSampleSize` (default 5 in Consul), use ALL_ZONE + region + day of week
- **Results stored in:** `route_tour_cost_stats` MongoDB collection
- Worker overwrites all existing records for the region when it runs

Default percentile config (configurable per region via Consul):

| Field | DB Field | Default Percentile |
|-------|----------|--------------------|
| Estimated Earnings | `average_earnings` | P50 |
| Most Drivers Earn (min) | `typical_earnings_min` | P20 |
| Most Drivers Earn (max) | `typical_earnings_max` | MAX |

Values are rounded to the nearest $0.50 and stored with precision up to 2 decimal places.

---

### 4.2 Earnings Range Sourcing — Pre-routing vs After Routing (MOB-2841)

Two outputs are computed independently: **Typical Earnings range (Most Drivers Earn)** and **Absolute Min–Max (Earnings Range)**. Each has a pre-routing (forecast) value and an after-routing value.

**Routing completion** is determined by the presence of assignments in `ticketBook.items` with `tourCost > 0`.

#### Typical Earnings range (Most Drivers Earn — P20 to MAX)

| Condition | Source |
|-----------|--------|
| Pre-routing, ≥ `minSampleSize` routes in zone-region-day | Historical P20–MAX; no change |
| Pre-routing, < `minSampleSize` routes, spread (MAX − P20) ≥ $10 | Historical P20–MAX; no change |
| Pre-routing, < `minSampleSize` routes, spread (MAX − P20) < $10 | Typical max = P20 + `minTypicalEarningsSpreadUsd` ($10), guaranteeing minimum $10 spread |
| Pre-routing, 0 routes (no data in `route_tour_cost_stats`) | Falls back to pay formula min → forecast max |
| After routing (any route count) | Actual routing min–max for zone |

#### Absolute Min–Max (Earnings Range)

| Field | Pre-routing | After routing |
|-------|-------------|---------------|
| **Min** | Pay formula min (WAT-2086 endpoint) | Pay formula min (WAT-2086 endpoint) |
| **Max** | Forecast max | Routing max (tour cost only) |

#### Average Earning (Estimated Earnings)

| Condition | Source |
|-----------|--------|
| Pre-routing | Historical P50 |
| After routing | Average tour cost from routing for zone |
| Value falls outside typical range | **Hidden** — not displayed |

#### Clamping Invariant

The typical range is always clamped within the absolute range:

- `typical_min ≥ absolute_min`
- `typical_max ≤ absolute_max`
- If `typical_min > typical_max` after clamping, `typical_min` is pulled down to equal `typical_max` (UI shows same value on both sides, e.g. $55 – $55)

---

### 4.3 Storage — Ticket Level (MOB-2541)

When a driver books a ticket, the three earnings fields are computed from `route_tour_cost_stats` and stored as a snapshot in the ticket MongoDB collection alongside the existing min/max pay fields.

- Fields added: `average_earnings`, `typical_earnings_min`, `typical_earnings_max` (DECIMAL, up to 2 decimal places)
- Values are **immutable** once stored — they do not change if the worker recalculates data later
- Fields are queryable in BigQuery
- Scope: used for total booked ticket + active ticket screens

---

### 4.4 Display — Booked & Active Screens (MOB-2517)

The Driver App FE reads the stored snapshot values from the ticket collection and displays them on:

- Total booked ticket screen
- Active ticket screen ("I'm arrived" state)

Display logic by region config:

| Condition | UI Behavior |
|-----------|-------------|
| Region in `enabledRegions` & data exists (count ≥ `minSampleSize`) | Show all 3 fields using zone-level data |
| Region in `enabledRegions` & count < `minSampleSize` | Show all 3 fields using ALL_ZONE fallback data |
| Region in `enabledRegions` but no data in DB | Fields hidden — no crash |
| Region NOT in `enabledRegions` | Only existing min/max range shown — no new fields, no crash |

---

### 4.5 Display — Booking Session Details Screen (MOB-2693)

The group card on the booking session (BS) details screen fetches values **in real-time** from `route_tour_cost_stats` on each page load. These are not snapshots.

The same display logic as 4.4 applies. Values are returned via the API under `decor_layout`.

**Key difference between BS details and booked ticket:**

- Group card (BS details page): fetched real-time from DB on each page load
- Booked ticket: snapshot taken at booking time, never recalculated

This means discrepancies can occur if Consul config changes or if the worker recalculates trailing data after a ticket was booked.

---

### 4.6 Code Changes (MOB-2841)

| Class / Component | Change |
|-------------------|--------|
| `EarningsRangeUtils` | New — central logic for computing typical range, absolute min/max, and average earnings across pre/post-routing states |
| `EarningsRange` | New — value object with clamping applied at construction; nulls out average earnings if outside typical range |
| `EstimatedEarningUtils` | Deleted — replaced entirely by `EarningsRangeUtils` |
| `getDriverEarningsMin` | Absolute min now sourced via `FinanceManager.evaluatePayFormula` (WAT-2086); does not differentiate by ticket type |
| `isRoutingCompleted` | Determined by presence of assignments in `ticketBook.items` with `tourCost > 0` |

---

## 5. Key Behaviors & Edge Cases

### 5.1 Data Freshness

- BS details group card: real-time fetch on each page load
- Booked ticket / active screen: snapshot taken once at booking time, never updated
- Pre-routing vs after-routing: values automatically switch once routing is confirmed (`tourCost > 0` on assignments)

### 5.2 Known Discrepancy Scenarios

Values on the BS details group card may differ from the booked ticket snapshot when:

- Consul config changes
- Worker recalculates trailing data after the ticket was originally created
- Routing completes after booking (group card reflects routing actuals; booked ticket holds the pre-routing snapshot)

### 5.3 Rounding

All three earnings values are rounded to **2 decimal places**.

### 5.4 Spread Floor

When the `minSampleSize` threshold is not met pre-routing and the historical spread (MAX − P20) < `minTypicalEarningsSpreadUsd` ($10), the typical max is set to P20 + $10 to guarantee a minimum visible spread.

### 5.5 Clamping Edge Cases

- If `typical_max` is clamped below `typical_min`, `typical_min` is pulled down to match `typical_max`
- If `average_earnings` (Estimated Earnings) falls outside the typical range after clamping, it is **hidden** from the UI

### 5.6 Rollout Strategy

| Ticket | Rollout |
|--------|---------|
| MOB-2518 (calculation worker) | Global — no region restriction |
| MOB-2517 / MOB-2541 / MOB-2693 (display) | Controlled via `enabledRegions` in Consul |
| MOB-2841 (refinements) | Same pattern — after PROD deploy, enable for all regions |

---

*Source: [Confluence](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2526707726/Estimated+Earnings+Typical+Driver+Earnings+Range+Documentation) · Last updated: 2026-06-29*
