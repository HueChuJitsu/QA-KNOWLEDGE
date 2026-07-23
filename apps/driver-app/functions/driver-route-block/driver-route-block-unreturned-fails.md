# Driver Route Block for Unreturned Fails & 30-Day Open Routes — QA Knowledge Document

*Blocks a driver from taking new routes while they still physically hold failed/unrouted packages, plus the extended open-routes window that lets those old routes stay visible so returns can be completed.*

**What you will learn**

- ✓ How the backend decides a driver is "blocked" (`GET /route-block`) and exactly which shipment states count as an outstanding item
- ✓ How the open-routes window was widened from ±2 days to −30/+2 days, and how the block gate reuses that same range
- ✓ The config/feature flags, resolution flow, edge cases, and reporting risks a tester must check before rollout

*Source: Jira MOB-2955 & MOB-3109 (Epic MOB-3108 "2026Q3 Unresolved Dropoff Fails")*

---

## 1. Overview

> When a driver marks a stop drop-off failed, they are prompted to **return** the package or **reattempt** delivery. Drivers sometimes dismiss both, leaving a stop stuck in `dropoff_failed` with no follow-up — the driver still physically holds the package. Separately, some shipments are pulled from a route (`unroute` event) and can no longer be marked failed/succeeded by the driver, but still need returning. This feature gives the Driver app a reliable server-side signal (`is_blocked`) so a driver cannot take a **new** route until those held packages are returned.
>
> Because the old route must stay visible for the return to be completed, MOB-3109 extends the open-routes window from ±2 days to **up to 30 days in the past** (+2 forward unchanged). The block gate (MOB-2955) reuses that same date range.

**Scope (QA):**

- `GET /route-block` endpoint behaviour: `is_blocked`, `outstanding_fail_count`, `offending_routes`.
- Which shipment/stop states count (or don't) as outstanding items.
- Open-routes list showing routes from −30 days to +2 days, region-flag gating, and performance.
- Identical behaviour across **DSP and 3P** routes, and **ticket (claimed)** and **direct-booking** flows.

**Who is affected:**

| Role | Impact |
|---|---|
| Driver | Blocked from taking a new route while holding unreturned failed/unrouted packages; must return them. Sees open routes up to 30 days old to complete returns. |

---

## 2. Key Terminology

| Term | Definition |
|---|---|
| **Outstanding item** | A shipment the driver still holds and must return. Counts when it has a **FAILED drop-off** stop **or** an **open RETURN** stop, AND no resolving follow-up (succeeded reattempt or succeeded return). |
| **`is_blocked`** | Boolean; `true` when ≥1 outstanding item exists for the driver. |
| **`outstanding_fail_count`** | Total **distinct** outstanding shipments (per-shipment dedup — same shipment held on two routes counts **once**). |
| **`offending_routes`** | List of `{ route_ref, count, stop_ids }`. `route_ref` = route label (falls back to route id if label null); `count` = distinct outstanding shipments on that route; `stop_ids` = offending stop ids. |
| **`dropoff_failed` / FAILED drop-off** | Stop status set when the driver fails a delivery attempt. |
| **Reattempt** | A follow-up drop-off stop (same `shipmentID`); a **succeeded** reattempt resolves the item. |
| **Return stop (`returnShipment`)** | Stop to return the package to the warehouse; a **succeeded** return resolves the item. Materialized by the MOB-2987 worker for unrouted/cancelled shipments. |
| **`driver_route_block`** | item_metadata `APP_CONFIG` value, format `"true/false"`. Resolved per region, warehouse and driver-app-api. If the flag hasn't been set up, `is_blocked` = `false`|
| **`launch_date` / `route_block_launch_date`** | Consul-configured floor date. Routes dated before it are never evaluated. Effective floor = `max(now − 30d, launch_date)`. If the flag hasn't set up, `is_blocked` = `false` |
| **`open_assignments_range`** | item_metadata `APP_CONFIG` value, format `"-30:2"` = (days back : days forward). Resolved per region and Consul. Regions that are not configured will be affected by Consul. |
| **DSP / 3P** | Delivery Service Provider routes vs third-party routes — behaviour must be identical. |

---

## 3. System / Flow Description

### Happy Path (block → resolve)

1. Driver fails a drop-off (`dropoff_failed`) on an in-scope route and dismisses both return & reattempt prompts.
2. `GET /route-block` (called as the driver) now returns `is_blocked = true`, the stop listed under `offending_routes[].stop_ids`, `count = 1`, `outstanding_fail_count = 1`.
3. Driver returns the package; outbound staff marks it `RETURNED` (succeeded return stop) — **or** a succeeded reattempt drop-off occurs.
4. `GET /route-block` reflects the change automatically (no manual refresh): the resolved stop drops off the list; when all clear, `is_blocked = false`, `outstanding_fail_count = 0`, `offending_routes = []`.

### Detection state machine (per shipment, on an in-scope route)

```text
[FAILED drop-off]  OR  [open RETURN stop (unroute/cancel)]
        │  (driver took possession, unresolved evidence)
        ▼
   OUTSTANDING  ──► is_blocked = true
        │
        ├── succeeded reattempt drop-off (same shipmentID) ──► RESOLVED (no block)
        └── succeeded return stop (returnShipment)         ──► RESOLVED (no block)
```

### Business Rules

- **In-scope window:** driver's routes with `predicted_start_ts ≥ launch_date`, excluding `COMPLETED`, over the resolved range (−30d … +2d). Route date evaluated in each **route's own local timezone**.
- **Outstanding condition (all must hold):** (1) driver took possession — a SUCCEEDED pickup OR a live open RETURN stop; (2) unresolved held evidence — a FAILED drop-off OR an open RETURN; (3) not delivered (no SUCCEEDED drop-off) and not returned (no SUCCEEDED return).
- **Not outstanding:** a merely **unattempted** stop (no `dropoff_failed`, no return stop / `unroute`) — endpoint short-circuits before the config read.
- **Failed reattempt** with no succeeded return → **still blocks**.
- **Dedup:** same shipment on two routes → counted once in `outstanding_fail_count`, but **both** routes listed in `offending_routes`.
- **Fail-safe:** missing/invalid `launch_date` → **not blocked** 
- **Flag priority:** `driver_route_block` resolves **warehouse → region → app-default (`driverappapi`)**; missing config → **not blocked**.
- **Tutorial shipments** are excluded from the count.
- **Open-routes range** resolution: per-region in item_metadata → Consul `driverappapi/open_assignments_range` → built-in default. Forward window stays +2 days.

---

## 4. Environment URLs & Access

**Config / flags:**

- `driver_route_block` — feature flag in item_metadata `APP_CONFIG` (Boolean), scoped by warehouse / region / `driverappapi` app-default.
- `route_block_launch_date` — Consul `apps.driverappapi.route_block.launch_date`.
- `open_assignments_range` — item_metadata `APP_CONFIG`, format `"-30:2"`, per region (auto-detected); Consul `driverappapi/open_assignments_range` fallback.
- `late_activation_hours` (Consul) → `720` hours (30 days).

**Required accounts / permissions:**

- [ ] Driver test account with routes across the −30d…+2d window (auth token → `user.externalId`).
- [ ] Driver with `dropoff_failed` / unrouted / return-pending shipments to trigger blocks.
- [ ] Access to set `driver_route_block` per warehouse/region and `launch_date` in Consul.
- [ ] Access to set `open_assignments_range` per region and `open_assignments_range` in Consul.
- [ ] Outbound/admin access to mark shipments `RETURNED` (to test resolution).
- [ ] Non-driver (admin) role to verify 403.

**Endpoint contract (`GET /route-block`):** no payload, no query params; driver identity from auth token; `@RolesAllowed("driver")`; produces `application/json` (snake_case). Non-driver → **403**.

```json
{
  "is_blocked": true,
  "outstanding_fail_count": 2,
  "offending_routes": [
    { "route_ref": "REF-A", "count": 1, "stop_ids": [100] },
    { "route_ref": "REF-B", "count": 1, "stop_ids": [200] }
  ]
}
```

---

## 5. Test Scenarios

### 5.1 Route-block detection — Happy Path (MOB-2955)

*Preconditions: `driver_route_block` ON for the route's warehouse/region (or app-default); `route_block_launch_date` = today or earlier; call `GET /route-block` as the driver.*

| # | Test Case | Steps | Expected Result | Priority |
|---|---|---|---|---|
| TC-01 | Failed drop-off, no follow-up | In-scope route with a `dropoff_failed` stop, no reattempt/return | `is_blocked = true`; stop in `offending_routes[].stop_ids`; `count = 1`; `outstanding_fail_count = 1` | High |
| TC-02 | Succeeded reattempt | `dropoff_failed` + succeeded reattempt drop-off (same shipment) | Not blocked | High |
| TC-03 | Succeeded return | `dropoff_failed` + succeeded RETURN stop | Not blocked | High |
| TC-04 | Unrouted shipment resolves | Unrouted shipment (worker RETURN stop = PENDING) → then scan return to SUCCEEDED | Blocks while pending; clears automatically after SUCCEEDED (no manual refresh) | High |
| TC-05 | `is_blocked` mapping | count = 0 → `is_blocked = false`; count = 1 → `is_blocked = true` | Correct boolean mapping | High |

### 5.2 Negative / Error & Boundary Cases

| # | Test Case | Steps | Expected Result | Priority |
|---|---|---|---|---|
| TC-06 | Failed reattempt still blocks | Failed reattempt, no succeeded return | Still blocks | High |
| TC-07 | Unattempted stop | Stop with no `dropoff_failed`, no `unroute`/return stop | Not blocked (short-circuits before config read) | High |
| TC-08 | Before launch_date | Route dated before `launch_date` with outstanding items | Ignored, not counted (e.g. launch_date = 2026-07-01, route 6/1 excluded) | High |
| TC-09 | Flag OFF at warehouse | `driver_route_block` = false for warehouse | Shipment not counted | High |
| TC-10 | Flag OFF at region | `driver_route_block` = false for region | Shipment not counted | High |
| TC-11 | No / invalid launch_date | `launch_date` unset or invalid | Feature off → not blocked; driver-route query not even issued | Medium |
| TC-12 | Auth 403 | Call `/route-block` as admin role | Returns **403** | High |
| TC-13 | Tutorial shipments | Driver has tutorial shipments in failed state | Not counted in `outstanding_fail_count` | Medium |

### 5.3 Dedup & Multi-route

| # | Test Case | Steps | Expected Result | Priority |
|---|---|---|---|---|
| TC-14 | Dedup — DF then reattempt fails | Driver `dropoff_fails` then reattempt fails (same shipment) | `outstanding_fail_count` counts it **once** | High |
| TC-15 | Dedup — DF then return pending/enroute | `dropoff_fails` then return pending/enroute | `outstanding_fail_count` counts it **once** | High |
| TC-16 | Multiple in-scope routes | Outstanding items on ≥2 in-scope routes | All counted; each route in `offending_routes`; same shipment on two routes counted once but both routes listed | Medium |
| TC-17 | route_ref fallback | Route with null label | `route_ref` shows route **id** | Low |

### 5.4 Open routes — 30-day window (MOB-3109)

| # | Test Case | Steps | Expected Result | Priority |
|---|---|---|---|---|
| TC-19 | See 30-day-old open route | Driver with open route up to 30 days in the past | Route appears in the hamburger menu; opening it brings up return-screen prompts | High |
| TC-20 | Forward window unchanged | Routes up to +2 days ahead | Still shown; routes at +3 days **not** shown | High |
| TC-21 | Older than 30 days | Open route 31+ days old | **Not** shown (boundary: `max(now−30d, launch_date) ≤ predicted_start_ts ≤ now+2d`) | High |
| TC-22 | COMPLETED excluded | Driver has `COMPLETED` routes | Never appear in the open list (excluded in SQL) | Medium |
| TC-23 | Region flag ON | item_metadata `open_assignments_range = "-30:2"` for a region | Drivers in that region see −30d…+2d | High |
| TC-24 | Unconfigured region | Region with no `open_assignments_range` (e.g. MCI) | Old (30-day) routes **not** displayed; Consul default used | High |
| TC-25 | Block reuses same range | ~20-day-old unreturned route vs route older than range | 20-day considered by block gate; older-than-range not | Medium |

---

## 6. Database Queries

```sql
WITH params AS (
  SELECT 311607::bigint AS driver_id, DATE '2026-07-22' AS launch_date   -- "from" date
),
routes AS (
  SELECT a.id, a.label, a.warehouse_id, a.region_code,
         (a.predicted_start_ts AT TIME ZONE 'UTC'
            AT TIME ZONE COALESCE(a.timezone,'America/Chicago'))::date AS route_date
  FROM assignments a, params p
  WHERE a.driver_id = p.driver_id
    AND a._deleted IS NOT TRUE
    AND (a.status IS NULL OR a.status <> 'COMPLETED')
    AND ('TUTORIAL' <> ALL(a.aggregated_tags) OR a.aggregated_tags IS NULL)
    AND a.predicted_start_ts >= (p.launch_date - 1)::timestamp
    AND a.predicted_start_ts <= now() + interval '1 day'
),
route_stops AS (
  SELECT s.id, s.shipment_id, s.assignment_id, s.type, s.status
  FROM stops s
  WHERE s.assignment_id IN (SELECT id FROM routes)
    AND s.type IN ('deliverShipment','returnShipment')
    AND s._deleted IS NOT TRUE
    AND s.shipment_id IS NOT NULL
),
evidence AS (                       -- held: only on routes dated >= the "from" date
  SELECT rs.*
  FROM route_stops rs
  JOIN routes r ON r.id = rs.assignment_id, params p
  WHERE r.route_date >= p.launch_date
    AND ( (rs.type = 'deliverShipment' AND rs.status = 'FAILED')
       OR (rs.type = 'returnShipment'  AND (rs.status IS NULL
                                            OR rs.status NOT IN ('SUCCEEDED','DISCARDED'))) )
),
resolved AS (                       -- succeeded reattempt/return anywhere in the window
  SELECT DISTINCT shipment_id
  FROM route_stops
  WHERE status = 'SUCCEEDED' AND type IN ('deliverShipment','returnShipment')
)
SELECT r.label AS route_ref, e.assignment_id AS route_id, e.shipment_id,
       e.id AS stop_id, e.type AS stop_type, e.status AS stop_status
FROM evidence e
JOIN routes r ON r.id = e.assignment_id
WHERE e.shipment_id NOT IN (SELECT shipment_id FROM resolved)
;
```
---

## 7. Related Tickets & References
Confluent: https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2649194499/Driver+route+block

---

