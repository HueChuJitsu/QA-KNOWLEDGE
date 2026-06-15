# Function: Delivery By

> **App:** Driver App · **Status:** Draft
> **Epic:** MOB-2514 — 2026Q2 Delivery By App Updates
> **Shared context:** see [../../README.md](../../README.md)
> **Config:** see [driver-otd-by-delivery-by.md](./driver-otd-by-delivery-by.md) — `enabled_regions_otd_calculated_by_delivery_by` Consul flag
> **Last updated:** 2026-06-08

## 1. Description
<!-- What the "Delivery By" function does, which actor it is for, and how it relates to the Driver App. -->

Deliver By is the feature that shows the delivery deadline to the driver. It has 3 components:

- **Delivery time window** (e.g. `7:00 AM – 7:00 PM`) shown on the **Pickup** and **Route Confirmation** screens.
- **Deliver By banner** shown persistently on the **active route** screen with ETD status (On Track / At Risk / Late).
- **OTD calculation** based on `delivery_by_ts` instead of `max_dropoff_latest_ts` (see [OTD spec](../../../webapp/dispatch-web/functions/driver-rating/otd.md)).

## 2. Business Flow
<!-- Main steps, entry & exit conditions — how "Delivery By" time is surfaced/used in the Driver App. -->

1. On the **Pickup** / **Route Confirmation** screens: show the delivery time window (static per region — see `delivery_window_by_region`).
2. After a successful pickup: compute `deliveryBy` (section 3.1) and show the **Deliver By banner** on the active route screen.
3. The banner auto-refreshes the ETD per `etd_refresh_interval` and updates the status (On Track / At Risk / Late — section 3.3).

### Banner Visibility Rules

| Condition | Banner shown? |
|-----------|--------------|
| `DELIVERY_enable_etd` = false | ❌ Hidden |
| `RG_{region}_DELIVERY_enable_etd` = false | ❌ Hidden |
| Multi-route assignment | ❌ Hidden |
| Tutorial assignment | ❌ Hidden (hard-coded) |
| Single route + `enable_etd` = true | ✅ Shown |

## 3. Spec / Rules
<!-- Business rules, validation, states -->

### 3.1 Delivery By Time — calculation

#### Single route

```
startTime     = pickup_succeeded_ts
totalTimeSec  = travelTime + serviceTime
bufferedSec   = totalTimeSec × (1 + bufferedPct)  // default bufferedPct = 1.3 → ×2.3
addSeconds    = ceil(bufferedSec)

rawDeliveryBy = startTime + addSeconds
deliveryBy    = roundUpToNearest5Min(rawDeliveryBy)
              → clamped to [earliestTime, latestTime]
```

#### Multiple routes (same-time pickup)

```
startTime     = MAX(pickup_succeeded_ts of all routes)  ← lastPickupTime
totalTimeSec  = SUM(travelTime + serviceTime of all routes)
bufferedSec   = totalTimeSec × (1 + bufferedPct)
addSeconds    = ceil(bufferedSec)

rawDeliveryBy = startTime + addSeconds
deliveryBy    = roundUpToNearest5Min(rawDeliveryBy)
              → clamped to [earliestTime, latestTime]
```

#### Clamping

| Condition | Result |
|---|---|
| `deliveryBy > latestTime` | use `latestTime` (default 8:00 PM) |
| `deliveryBy < earliestTime` | use `earliestTime` (default 7:00 AM) |

**Fallback:** if there is no ETA or the feature is disabled → `7:00 PM` (device local time).

### 3.2 ETD — calculation

#### Single route (mobile)

```
ETD = now + remainingDeliveryTime
```

#### 2 routes (mobile)

```
ETD = fetchedAt + RDT(1) + transitRouteTime + RDT(2)
```

#### `remaining_delivery_time_in_seconds` (server-side)

```
First stop:        stopTime = max(0, estimatedArrivalTs - now) + serviceTime
Remaining stops:   stopTime = travelTimeFromPreviousStop + serviceTime

remaining_delivery_time_in_seconds = SUM(stopTime) over all remaining DROP_OFF stops
```

Query filter:

```sql
type = 'DROP_OFF'
AND status NOT IN ('SUCCEEDED', 'FAILED', 'CANCELLED')
ORDER BY sequence_id ASC
```

### 3.3 Status Logic

```
On Track : predictedEnd < deliveryByTime - threshold
At Risk  : deliveryByTime - threshold ≤ predictedEnd ≤ deliveryByTime
Late     : predictedEnd > deliveryByTime
```

| Status | Color | Hex |
|--------|-------|-----|
| 🟢 On Track | Green | `#297a48` |
| 🟡 At Risk | Orange | `#f97500` |
| 🔴 Late | Red | `#ba1a1a` |
| ⚫ No ETA | Grey | — |

### 3.4 Consul Config Keys

| Key | Service | Default | Restart | Description |
|-----|---------|---------|---------|-------------|
| `prod/apps/driverappapi/delivery/DELIVERY_enable_etd` | driver-app-api | `false` | ✅ Yes | Global enable ETD banner |
| `prod/apps/driverappapi/delivery/RG_{regionCode}_DELIVERY_enable_etd` | driver-app-api | `false` | ✅ Yes | Per-region enable ETD banner |
| `prod/apps/driverappapi/delivery/deliver_by_at_risk_threshold` | driver-app-api | `15` (min) | ❌ No | At Risk threshold |
| `prod/apps/driverappapi/delivery/delivery_window_enabled_multiple_routes` | driver-app-api | `false` | ⚠️ Confirm | Enable multiple-routes delivery window |
| `prod/apps/driverappapi/delivery/delivery_window_by_region` | driver-app-api | `8AM-7PM` | ✅ Yes | Static delivery window per region (Pickup & Confirmation screens) |
| `prod/apps/driverappapi/delivery/notice_on_combined_delivery_time` | dispatch-bizlogic, driver-app-api | `false` | ✅ Yes | Show popup notice when a combined window applies |
| `prod/apps/driverappapi/mobile_app_config/etd_refresh_interval` | driver-app-api | `5` (min) | ⚠️ Confirm | ETD auto-refresh interval |
| `apps/dispatch-bizlogic/delivery_window/same_time_pickup_threshold_minutes` | dispatch-bizlogic | `60` (min) | ⚠️ Confirm | Same-time pickup threshold |

### 3.5 Data Model

#### `estimated_delivery_window` — primary source of `delivery_by_time`

| Field | Type | Description |
|-------|------|-------------|
| `assignment_id` | Long | Main route |
| `driver_id` | Long | Driver who owns the window |
| `warehouse_id` | Long | Lookup scope |
| `same_time_pickup_ids` | List\<Long\> | Sibling assignment IDs |
| `delivery_by_ts` | Long | Unix ms deadline |
| `delivery_time_window` | String | e.g. `"7:00 AM - 4:05 PM"` |

#### `stops` — used to compute `remaining_delivery_time_in_seconds`

| Field | Type | Description |
|-------|------|-------------|
| `assignment_id` | Long | Route of the stop |
| `sequence_id` | Int | Stop order |
| `type` | String | Filter: `DROP_OFF` only |
| `status` | String | Exclude: `SUCCEEDED`, `FAILED`, `CANCELLED` |
| `service_time` | Double | Seconds to complete the stop |
| `travel_time_from_previous_stop` | Double | Travel time from the previous stop |
| `estimated_arrival_ts` | DateTime | Realtime ETA (updated from GPS) |

## 4. QA / Test notes
<!-- Happy cases, edge cases, sample data, things to watch when testing -->

### Edge cases / things to watch

- **Tutorial route:** banner is hard-coded to hide, the ETD API returns an empty response — **cannot be overridden by config**.
- **DSP route (`courierId != null`):** not supported yet, no timeline (Open Item in MOB-2449).
- **3+ routes:** `transitRouteTime = 0` (inaccurate) — Open Item in MOB-2449.
- **`delivery_window_by_region`:** static config for the Pickup & Confirmation screens — **not** affected by the MOB-2237 dynamic calculation.
- **Multiple routes:** only the **Global** switch is supported, no per-region support.
- **`stops.status`:** the MOB-2449 spec incorrectly says `COMPLETED` — the actual value is `SUCCEEDED`.
- **When multiple routes is not enabled:** routes without an ETA always fall back to `7:00 PM`.

### Suggested cases to cover

- **Status boundary:** `predictedEnd == deliveryByTime - threshold` (On Track / At Risk boundary) and `predictedEnd == deliveryByTime` (At Risk / Late boundary).
- **Clamping:** `deliveryBy` exceeding `latestTime` (8:00 PM) and below `earliestTime` (7:00 AM).
- **Fallback:** no ETA / feature off → window `7:00 PM`.
- **Banner visibility:** cover all 5 conditions in section 2 (global off, region off, multi-route, tutorial, single-route on).
- **Multiple routes:** `startTime` = MAX(pickup), `totalTimeSec` = SUM of all routes.

## 5. Related Jira & References
<!-- Link the epic / tickets for this function. -->

### Jira Tickets

| Ticket | Summary | Status |
|--------|---------|--------|
| MOB-2514 | 2026Q2 Delivery By App Updates (**Epic**) | In Progress |
| MOB-2534 | [FE] Show dynamic Deliver By banner + on-track progress | Staging Review |
| MOB-2597 | [BE] Add stop info for 2 routes (ETD API) | Done |
| MOB-2714 | [FE] Deliver By banner + on-track progress for 2 routes | Staging Review |
| MOB-2449 | Calculate ETD for DELIVER_BY for 2 routes | Rollout Ready |
| MOB-2237 | Add new field DELIVER_BY for 2 routes | Rollout Ready |
| MOB-2521 | Update Driver Ratings OTD Calculation to use DELIVER BY time | Done |
| MOB-2530 | Configurable same-time pickup threshold | Done |
| MOB-2528 | Re-call traffic-info on accept route | — |
| MOB-467 | Add new field DELIVERY_BY for single route | Rollout Ready |

### Confluence Documentation

| Page | Link |
|------|------|
| 📁 Delivery by and ETD *(parent)* | [Delivery by and ETD](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2503245856/Delivery+by+and+ETD) |
| 📄 Delivery Time Window — Single Route Spec | [DELIVERY TIME WINDOW – SINGLE ROUTE SPECIFICATION](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2312470548) |
| 📄 Delivery Time Window — Multiple Routes Spec (MOB-2237) | [DELIVERY TIME WINDOW – MULTIPLE ROUTES SPECIFICATION](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2447933443) |
| 📄 Deliver By Banner Config (MOB-2534) | [Deliver By Banner Config](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2503901190) |
| 📄 Remove Deliver By from Booking Session Level | [Remove 'Deliver By' from Booking Session Level – Configuration](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2395045896) |
| 📄 Show delivery expectation / delivery window on driver app | [Show delivery expectation](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1297645590) |
