# One-Off Sprinkle — Data Setup, Config Checks & Scan Flow

> **ℹ️ INFO**
> Reference / onboarding page for QA, QI and Devs. Explains what one-off sprinkle is, where the data & config live, and how scanning a shipment (**small** vs **large**) does — or does not — enter the one-off sprinkle flow. Includes a playbook to reproduce each outcome.
>
> **Context:** `ALT-1883`  ·  **Env:** `staging`  ·  **Example region:** `JFK / client 980`  ·  **Core lib:** `sortation-bizlogic`

**Labels:** `routing` `sortation` `sprinkling` `qa-runbook` `ALT-1883`

---

## On this page

1. [What one-off sprinkle is](#1-what-one-off-sprinkle-is)
2. [Where code & data live](#2-where-code--data-live)
3. [The two scan paths](#3-the-two-scan-paths)
4. [The gates in detail](#4-the-gates-in-detail)
5. [Data setup checklist](#5-data-setup-checklist)
6. [Config checks](#6-config-checks)
7. [Delivery-date formula](#7-delivery-date-formula)
8. [Playbook — reproduce each outcome](#8-playbook--reproduce-each-outcome)
9. [Gotchas](#9-gotchas)
10. [Reference queries](#10-reference-queries)
11. [Decision flow](#11-decision-flow)

---

## 1. What one-off sprinkle is

When a shipment that is **not yet routed** (or a **failed delivery needing redelivery**) is scanned at the warehouse, the system tries to **insert it into a nearby running route/assignment** — that is *one-off sprinkle*.

- Core code: `SprinklingManager.doOneOffSprinkleV2(...)` in the **`sortation-bizlogic`** lib.
- On failure it emits `SHIPMENT.PLANNING.one-off-failed` with a **reason** (`OneOffFailure` enum) and writes a **trace** to the Mongo collection `sprinkling_traces`.

> **💡 TIP — Golden debugging rule**
> No document in `sprinkling_traces` for the shipment → `doOneOffSprinkleV2` **never ran** — it was blocked **before** the one-off call (at the warehouse-api / inbound-api scan layer).

**`OneOffFailure` — main reasons**

```text
INVALID_STATUS, CLIENT_NOT_SPRINKLABLE, SPRINKLING_CONFIG_NOT_FOUND,
TOO_EARLY, BUSINESS_CLOSED_ON_DELIVERY_DATE,        // ALT-1883 added the latter
NO_NEAREST_STOP, NEAREST_ASSIGNMENTS_OVER_LOADED, NEAREST_ASSIGNMENTS_NOT_SATISFY, ...
```

---

## 2. Where code & data live

### Repositories

| Repo | Role |
| --- | --- |
| `sortation-bizlogic` | Lib: `SprinklingManager` (checkDate, doOneOffSprinkleV2), `SortInfoManager` (large scan), `OneOffFailure` |
| `warehouse-api` | **Small-parcel** scan flow (WipService, SmallParcelSessionService, SmallSortScanHelper) |
| `inbound-api` | **Large** scan flow (SortInfoRS → SortInfoManager.scanInbound); also exposes gRPC `oneOffSprinkle` |
| `inbound-api-grpc` | Receives gRPC from warehouse-api → calls lib `oneOffSprinkleV2` |
| `routing-bizlogic` | `BusinessHourManager` (business hours, getDeliveryEndTime, canDeliverShipmentOnDate) |
| `worker` | DeliverableAddressWorker (DAS event), ShipmentBusinessHourHandler (writes `shipment_extra`) |

> **ℹ️ NOTE**
> The `oneOffSprinkle` gRPC in **inbound-api-grpc** uses the `com.axlehire` lib `SprinklingManager` (which has the ALT-1883 change) — **not** the forked `com.jitsu.manager` copy that also lives in that repo.

### Data — Postgres vs MongoDB

| Data | Store | Table / Collection | Key fields |
| --- | --- | --- | --- |
| Shipment (core) | **Postgres** | `shipments` | status, assignment_id, customer_profile_id, volumic, inbound_status, inbound_received_ts, service_level |
| Business flag | **Postgres** | `shipment_extra` | **is_business**, routing_*_dropoff_ts |
| Stops | **Postgres** | `stops` | type (deliverShipment / pickupShipment), status |
| Assignment (route) | **Postgres** | `assignments` | label, vehicle_id, volumic, shipment_count, solution_id, skills |
| Client service | **Postgres** | `client_service` | **sprinklable**, enable_business_hour, region |
| Route prefix | **Postgres** | `route_prefixes` | capacity, max_box, **sprinkle** |
| Business hours | **Mongo** | `deliverable_address` | **business**, **business_hours** (day-of-week map) |
| Profile → address | **Mongo** | `customer_profile` | deliverable_address_id |
| Sprinkle config | **Mongo** | `sprinkling_config` | regions, attributes.sort_start/sort_end, limit_by_vehicle, redelivery_validator |
| Sprinkle trace | **Mongo** | `sprinkling_traces` | failure_reason, target_delivery_date, failure_context |
| Routing solution | **Mongo** | `routing_export_solution` | vehicle_map (assignmentId → vehicle/capacity) |
| Drools rule | **Mongo** | `drl` | type = delivery_validator_staging, field `rules` |

> **⚠️ WARNING**
> Shipment core + assignment + stops + client_service are in **Postgres**, while business hours + sprinkle config are in **Mongo**. Debugging almost always means **joining the two stores by hand**.

---

## 3. The two scan paths

### Small parcel (warehouse-api)

```text
Scan → warehouse-api (WipService / SmallParcelSessionService / SmallSortScanHelper)
   → gRPC oneOffSprinkle → inbound-api-grpc → lib oneOffSprinkleV2 → checkDate
```

One-off is called when the shipment is **not routed** (`isRouted == false`) and passes the gates.

### Large package (inbound-api)

```text
Scan (large) → inbound-api SortInfoRS → SortInfoManager.scanInbound
   → if (processInbound && canAutoRoute) → getBusinessHourLabel guard → oneOffSprinkleV2
```

### Comparison — the core difference

| | Small parcel | Large (inbound scan) |
| --- | --- | --- |
| Route condition to sprinkle | `isRouted == false` | `canAutoRoute == true` (requires `sortType == FINAL`) |
| **Business-closed guard** | `if (assignmentId == null && isClosed)` | `if (!isOpened)` — **ignores assignmentId** |
| Redelivery (assignmentId≠null) + business closed | **BYPASSES guard** → enters sprinkle → `BUSINESS_CLOSED_ON_DELIVERY_DATE` | **Still blocked** (prints business ZPL) |
| No-route (assignmentId==null) + business closed | Blocked ("Business closed do not sort") | Blocked |

> **⚠️ WARNING — Key consequence**
> The `BUSINESS_CLOSED_ON_DELIVERY_DATE` reason (from the sprinkle path) is essentially **only reproducible via the small-parcel redelivery path** (assignmentId≠null). On the large path, a closed business is always intercepted first, so it never reaches one-off.

---

## 4. The gates in detail

> Evaluated in order. Only after all of them do we reach `checkDate`, where `TOO_EARLY` / `BUSINESS_CLOSED` are produced.

### A · Shipment state (validateShipment + scan layer)

Rejected (no sprinkle) if:

- Flags `is_cancelled / is_disposable / is_undeliverable / is_unroutable / is_returning_to_client = true`
- `status` ∈ `shouldNotRouteStatuses` = {DAMAGED, RETURN_DAMAGED, DISPOSABLE, UNDELIVERABLE, RETURNED_TO_CLIENT_SUCCEEDED, RETURNED_TO_CLIENT_PENDING, INVALID_ADDRESS, UNSERVICEABLE, GEOCODE_FAILED}
- `inbound_status` ∈ {RECEIVED_DAMAGED, RECEIVED_LEAKING} — so `RECEIVED_OK` is **not** blocked

### B · Route state (isRouted / canAutoRoute)

Only **DROP_OFF + RETURN** stops are considered (pickup ignored). Treated as *not routed → sprinkle* when:

```text
stopList is empty
  OR ( every DROP_OFF ∈ {FAILED, DISCARDED}  AND  every RETURN = SUCCEEDED )
```

- `assignment_id == null` → not routed → sprinkle.
- Every DROP_OFF already `SUCCEEDED` → DeliveredShipment (already delivered, no sprinkle).
- Large also requires `transition.sortType == FINAL` (PRESORT → canAutoRoute = false).

### C · Business-hours guard (differs per path — see §3)

`BusinessHourManager` decides open/closed from `shipment_extra.is_business` (Postgres) + `deliverable_address.business_hours[<dayOfWeek>].status` (Mongo, compared to `OPEN_STATUS = "Open"`) + no-service-date / weekend. If `is_business = false` → always treated as open.

### D · Client service

`sprinklable == true` (false → `CLIENT_NOT_SPRINKLABLE`). `enable_business_hour` must be on for business hours to apply.

### E · checkDate — where TOO_EARLY & BUSINESS_CLOSED are produced

1. **TOO_EARLY** (checked FIRST): `start.plusDays(days).isAfter(now) && inboundReceivedTs != null`
2. **BUSINESS_CLOSED_ON_DELIVERY_DATE**: `getDeliveryEndTime()` returns empty (business closed on delivery date)
3. OK → continue to nearest-neighbor → overload

### F · Overload (NEAREST_ASSIGNMENTS_OVER_LOADED)

JFK uses the Drools rule `delivery_validator_staging` (config points to it via `redelivery_validator`):

- Effective capacity = `max(vehicleCategory.capacity, routePrefix.capacity)`
- `vehicleCategory` comes from `routing_export_solution.vehicle_map[assignmentId]`; **missing → the assignment is dropped**
- No assignment survives → `NEAREST_ASSIGNMENTS_OVER_LOADED`

---

## 5. Data setup checklist

> Goal: a shipment that actually **enters** one-off sprinkle. The most reliable setup is a **small-parcel redelivery**.

### Postgres

- [ ] `shipments`: region/client in a configured region (e.g. JFK/980); status not in shouldNotRouteStatuses; `inbound_status = RECEIVED_OK`.
- [ ] Redelivery: has `assignment_id`, and in `stops` the drop-off (`deliverShipment`) = `FAILED`.
- [ ] `inbound_received_ts != null` (required to test TOO_EARLY).
- [ ] `shipment_extra.is_business = true` if testing business hours (see Gotcha #1).
- [ ] `client_service`: `sprinklable = true`, `enable_business_hour = true`.

### Mongo

- [ ] `customer_profile[cp_id].deliverable_address_id` points to the right DA.
- [ ] `deliverable_address[da].business = true` and `business_hours[<day>].status` correct (Open/Closed).
- [ ] `sprinkling_config` (regions contains the region) exists & is active.

### Reference examples (built during the investigation)

| Shipment | Scenario | Outcome |
| --- | --- | --- |
| `70633002` | redelivery small, is_business=true, Tue=Closed | **`BUSINESS_CLOSED`** |
| `70633504` | redelivery, business open | **`OVERLOADED`** |
| `70632956` | already routed (assignment still active) | *not sprinkled* |
| `70637655` | large scan, target WED=Closed | *does not enter one-off* |

---

## 6. Config checks

### sprinkling_config (Mongo) — JFK example

```jsonc
{
  "regions": ["JFK"],
  "_etag": "b5218e7c-...",          // = sprinkling_config_id in the trace
  "attributes": {
    "sort_start": "PT12H",           // opens the sort window (affects TOO_EARLY)
    "sort_end":   "PT36H",           // determines "days" & delivery date
    "redelivery_validator": "delivery_validator_staging"  // rule in the `drl` collection
  },
  "limit_by_vehicle": { "SEDAN": {"max_capacity":30,"max_box_count":40}, ... }
}
```

- `route_prefixes` (Postgres): `sprinkle=true` (false → route excluded from candidates); `capacity` / `max_box` = workload / parcel-count ceilings.
- `client_service` (Postgres): `sprinklable`, `enable_business_hour`.
- Drools `delivery_validator_staging` (`drl`): overload logic. **Editing the rule needs a cache flush / restart** of the service running sprinkle.

---

## 7. Delivery-date formula

> **ℹ️ NOTE**
> `service_level` (NEXT_DAY / SAME_DAY) is **NOT** used here. The delivery date is derived from `sort_start/sort_end` + scan time.

```text
date0 = startOfDay(now, tz) − 1 day
start = date0 + sort_start          // Period, e.g. PT12H
end   = date0 + sort_end            // e.g. PT36H
days  = min d≥0 such that  end + d·24h ≥ now      // → depends on sort_end only
target_delivery_date = date0 + days

TOO_EARLY  ⇔  (start + days·24h) > now   AND  inbound_received_ts != null
```

In plain terms: **TOO_EARLY = sprinkling before the sort window for the delivery day opens.** Raise `sort_start` so the window opens later than the scan time; keep `sort_end` unchanged (does not change days / delivery date).

---

## 8. Playbook — reproduce each outcome

### ✅ SUCCESS
Redelivery small parcel, business **open** on the delivery date, a nearby assignment with room (enough capacity, vehicle_map has a mapped vehicle).

### 🔴 BUSINESS_CLOSED_ON_DELIVERY_DATE
- **Small-parcel redelivery** (assignment_id≠null, drop-off FAILED).
- `shipment_extra.is_business = true`; `business_hours[<delivery day>].status = "Closed"`; DA linkage correct.
- Scan on a day the business is closed.

> **⚠️ WARNING**
> Do **not** use the large scan (business guard blocks first) and do **not** use a no-route business address (blocked with "Business closed do not sort").

### 🔴 TOO_EARLY
- Raise `attributes.sort_start` so the window opens later than the scan time (e.g. `PT12H → PT34H`). Keep `sort_end`.
- Requires `inbound_received_ts != null`.
- **Redelivery**: yields TOO_EARLY regardless of business open/closed (guard bypassed + too-early is checked before the business check).
- **No-route**: must pass the business guard first — shipment not a business address OR delivery day Open.

### 🔴 NEAREST_ASSIGNMENTS_OVER_LOADED
There are nearby candidates but the validator won't insert: capacity full, OR `routing_export_solution.vehicle_map` is missing the assignment (→ vehicleCategory null → dropped), OR route-prefix `sprinkle = false`.

### ⚪ NO TRACE (does not enter the flow)
Already routed (isRouted=true) / already delivered (DeliveredShipment) / bad status / regional / excluded client / large: `sortType != FINAL` / business guard blocks (large, or no-route business closed).

---

## 9. Gotchas

> **🔴 1 · is_business is NOT set from address_type**
> `shipment_extra.is_business` is written by the `ShipmentBusinessHourHandler` worker when it processes a DAS event (DAS-success/corrected/flagged), taking the flag from geocoding (residential=false), and **only when business-hour is enabled** for the region/client. Manually setting `address_type = COMMERCIAL_BUILDING` does **not** flip it. Fast path: upsert `shipment_extra` directly (Postgres).

> **ℹ️ 2 · Two different sources**
> `is_business` (Postgres `shipment_extra`) vs `business_hours` (Mongo `deliverable_address` via `customer_profile`). A mismatch between the two causes wrong behavior.

> **ℹ️ 3 · Business-closed guard differs**
> Small blocks only when `assignmentId == null`; large blocks **always**.

> **ℹ️ 4 · service_level does not affect `determineDeliveryDate`**
> The delivery date is driven by sort_start/sort_end + scan time.

> **ℹ️ 5 · sort_end vs sort_start**
> `sort_end` determines days / delivery date; `sort_start` determines the TOO_EARLY threshold (does not touch days).

> **🔴 6 · Overload can be "fake"**
> If `routing_export_solution.vehicle_map` is missing the candidate's vehicle → vehicleCategory null → dropped → labeled OVERLOAD even though real capacity is plentiful.

> **ℹ️ 7 · Config/rules are cached**
> RMapCache, Drools session — after editing Mongo/Postgres you may need to wait for TTL / restart.

> **ℹ️ 8 · Trace is the source of truth**
> `sprinkling_traces` filtered by `shipment_id` (snake_case). No trace = never entered one-off.

---

## 10. Reference queries

### Postgres

```sql
-- Shipment state + service level
SELECT id, status, region_code, client_id, assignment_id, customer_profile_id,
       volumic, inbound_status, inbound_received_ts, service_level
FROM shipments WHERE id = :id;

-- Business flag
SELECT * FROM shipment_extra WHERE id = :id;

-- Stops (route state)
SELECT type, status FROM stops WHERE shipment_id = :id AND _deleted IS NOT TRUE;

-- Client service
SELECT sprinklable, enable_business_hour FROM client_service
WHERE client_id = :clientId AND region = :region AND _deleted IS NOT TRUE;

-- Quick is_business upsert (for testing)
INSERT INTO shipment_extra (id, is_business, routing_earliest_dropoff_ts, routing_latest_dropoff_ts)
VALUES (:id, true, :earliest_ts, :latest_ts)
ON CONFLICT (id) DO UPDATE SET is_business = true, _updated = NOW();
```

### Mongo

```js
// Sprinkle result trace
db.sprinkling_traces.find({ shipment_id: <id> }).sort({ _id: -1 })

// Region config
db.sprinkling_config.find({ regions: "JFK" })

// Business hours of the DA (da_id via customer_profile.deliverable_address_id)
db.customer_profile.find({ id: "<cp_id>" }, { deliverable_address_id: 1 })
db.deliverable_address.find({ id: "<da_id>" }, { business: 1, business_hours: 1 })

// Solution vehicle map (explains overload)
db.routing_export_solution.find({ solution_id: <id> }, { vehicle_map: 1 })
```

---

## 11. Decision flow

```text
SCAN shipment
  ├─ bad status / cancelled / geocode-failed ....................... STOP (no sprinkle)
  ├─ regional / excluded client ................................... STOP
  ├─ [LARGE] sortType != FINAL .................................... STOP (canAutoRoute=false)
  ├─ already routed (isRouted=true) / delivered ................... STOP
  ├─ business CLOSED:
  │     • LARGE .................................................. STOP (prints business ZPL)
  │     • SMALL no-route (assignmentId==null, is_business) ....... STOP ("Business closed do not sort")
  │     • SMALL redelivery (assignmentId!=null) .................. CONTINUE (bypasses guard)
  ├─ call oneOffSprinkleV2 → checkDate:
        ├─ before sort_start & inbound_received_ts!=null .......... TOO_EARLY
        ├─ business closed on delivery date ...................... BUSINESS_CLOSED_ON_DELIVERY_DATE
        ├─ no assignment has room / missing vehicle map .......... NEAREST_ASSIGNMENTS_OVER_LOADED
        └─ insertable ........................................... SUCCESS
```

---

> *Compiled from a staging investigation (ALT-1883). Mechanics described against the current source HEAD of `sortation-bizlogic`; the example shipments/config reflect staging at the time of writing.*
