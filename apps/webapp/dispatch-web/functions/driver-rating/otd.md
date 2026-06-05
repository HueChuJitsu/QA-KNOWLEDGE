# Function: OTD (On-Time Delivery)

> **App:** Dispatch Web · **Part of:** Driver rating · **Status:** Draft
> **Shared context:** see [../../README.md](../../README.md)
>
> **Source:** `rating-bizlogic` → `DriverRatingCalculator.calculateOnTimeDeliveryRate()`
> **Collection:** `driver_rating_otd` (MongoDB)

## 1. Description
<!-- What the OTD (On-Time Delivery) metric is, how it relates to Driver rating, and which actor it is for. -->

OTD (On-Time Delivery) measures the share of a driver's deliveries completed on time. It is one of the components of the **Driver rating**. OTD value = number of `ONTIME` deliveries / total valid deliveries (see section 3.7).

## 2. Business Flow
<!-- How OTD is measured/collected, entry & exit conditions. -->

OTD is computed from the `driver_rating_otd` collection. Each record is a single delivery; a record is filtered (section 3.1 — *Record Filter*), classified by `isNextDay`, has its `endTs` / `deliveryByTs` timestamps computed, passes the *Gate Check*, and is then classified as `ONTIME` / `EARLY` / `LATE`. See the full flow in [section 3.8](#38-flow-diagram).

## 3. Spec / Rules
<!-- OTD definition, on-time threshold, calculation window, how it feeds into the rating. -->

### 3.1 Record Filter

Only records matching **all** of the following conditions are included in the calculation:

| Field | Condition |
|-------|-----------|
| `pickup_status` | `SUCCEEDED` |
| `dropoff_status` | `SUCCEEDED` or `FAILED` |
| `_deleted` | `false` |
| `start_delivery_ts` | must exist |
| `end_delivery_ts` | must exist |
| `dropoff_ts` | must exist |

> ✅ **Confirmed:** when `dropoff_status = FAILED`, `dropoff_ts` **is still set** → FAILED records pass the filter and are classified by OTD normally (by design).

### 3.2 Determine `isNextDay`

```
isNextDay = true when:
  service_type = COMMINGLE AND service_level IN [NEXT_DAY, ""]
  service_type = ONDEMAND  AND service_level IN [NEXT_DAY, ""]   ← see section 3.6

isNextDay = false for all other combinations
```

> ✅ **Confirmed:**
> - The `ONDEMAND + NEXT_DAY` branch has been **released** — this is the **current production logic**, no longer a pending fix.
> - `service_level = ""` (empty) **is treated as `NEXT_DAY`** (intended, not a bug).

### 3.3 Calculate `endTs`

```
endTs = max_dropoff_latest_ts        (or 0 if null)

Only when isNextDay = true (COMMINGLE + NEXT_DAY or ONDEMAND + NEXT_DAY):
  estimatedTime = end_delivery_ts - start_delivery_ts
  endTime       = start_delivery_ts + estimatedTime × (1 + buffer)
  if endTime > endTs → endTs = endTime
```

> ✅ **Confirmed:**
> - `buffer` is configured by a **formula in the `driver_rating_config` table**, differentiated by driver type **IC** and **DSP**. Default: `0%`.
> - The `endTime` (buffer) computation applies to **both** COMMINGLE + NEXT_DAY **and** ONDEMAND + NEXT_DAY (i.e. all cases where `isNextDay = true`).

### 3.4 Determine `deliveryByTs`

```
shipmentId IN estimated_delivery_window.specialty_client_shipment_ids ?

  YES → deliveryByTs = endTs          (the endTs computed in section 3.3 — NOT always = max_dropoff_latest_ts)
  NO  → deliveryByTs = estimated_delivery_window.delivery_by_ts
        (fallback to endTs if no window record exists)
```

> **Specialty client IDs** are configured via the Consul key `apps.driverappapi.delivery.specialty_client_ids`, and stored in `estimated_delivery_window.specialty_client_shipment_ids` per assignment.

### 3.5 Gate Check

A record is **skipped** (not counted) if any condition fails:

```
deliveryByTs > 0
min_dropoff_earliest_ts != null
dropoff_ts != null        (defensive — already guaranteed by the filter in section 3.1)
```

### 3.6 OTD Classification

```
dropoff_ts < min_dropoff_earliest_ts  → EARLY branch:
  ├── COMMINGLE + NEXT_DAY → ONTIME ✅
  ├── ONDEMAND  + NEXT_DAY → LATE   ❌   (delivered early still counts as LATE — see note)
  └── others               → EARLY  (denominator only, not in numerator)

dropoff_ts > deliveryByTs             → LATE ❌

otherwise                             → ONTIME ✅
```

**Boundary (exactly on the boundary = ONTIME):**
- `dropoff_ts == min_dropoff_earliest_ts` → **not** EARLY (uses `<`) → continue to the lower branch.
- `dropoff_ts == deliveryByTs` → **not** LATE (uses `>`) → `ONTIME`.

> ✅ **Confirmed:** for ONDEMAND + NEXT_DAY, an **early** delivery still counts as `LATE` — this is **intended logic** (not a bug).

### 3.7 OTD Value

```
numerator   = onTime count
denominator = onTime + early + late      (note: the "EARLY others" from section 3.6 are NOT in the denominator)
OTD %       = numerator / denominator
```

> - If `denominator = 0` → the rate cannot be computed; return the "insufficient data" value (see line below).
> - If total records < the `min` threshold → OTD value = `records - min` (negative, meaning insufficient data).
>
> ✅ **Confirmed:** the `min` threshold is taken from the **latest `min` value in the `driver_rating_config` table**.

### 3.8 Flow Diagram

```
driver_rating_otd records
  pickup_status = SUCCEEDED
  dropoff_status IN [SUCCEEDED, FAILED]
  _deleted = false
        │
        ▼
  isNextDay?
    COMMINGLE + NEXT_DAY → true
    ONDEMAND  + NEXT_DAY → true (released)
    others               → false
        │
        ▼
  endTs = max_dropoff_latest_ts
    if isNextDay (COMMINGLE + NEXT_DAY or ONDEMAND + NEXT_DAY):
      endTime = start_delivery_ts + estimatedTime × (1 + buffer)
      if endTime > endTs → endTs = endTime
        │
        ▼
  deliveryByTs?
    ├── specialty (in specialty_client_shipment_ids)
    │     → deliveryByTs = endTs   (the endTs computed above)
    └── non-specialty
          → deliveryByTs = estimated_delivery_window.delivery_by_ts
            (fallback: endTs)
        │
        ▼
  Gate: deliveryByTs > 0
        && minDropoffEarliestTs != null
        && dropoffTs != null
        │
   FAIL─┤─PASS
   SKIP │
        ▼
  dropoff_ts < min_dropoff_earliest_ts?
    ├── YES (early):
    │     ├── COMMINGLE + NEXT_DAY → ONTIME ✅
    │     ├── ONDEMAND  + NEXT_DAY → LATE   ❌
    │     └── others               → EARLY (denominator only)
    └── NO:
          ├── dropoff_ts > deliveryByTs → LATE ❌
          └── otherwise                 → ONTIME ✅
        │
        ▼
  OTD % = onTime / (onTime + early + late)
```

### 3.9 Key Fields Reference

#### `driver_rating_otd` fields used in the calculation

| Field | Role |
|-------|------|
| `service_type` | Determine `isNextDay` |
| `service_level` | Determine `isNextDay` |
| `start_delivery_ts` | Calculate `endTime` (when `isNextDay = true`) |
| `end_delivery_ts` | Calculate `endTime` (when `isNextDay = true`) |
| `max_dropoff_latest_ts` | Base `endTs` → fallback `deliveryByTs` for specialty |
| `min_dropoff_earliest_ts` | Early check boundary |
| `dropoff_ts` | Compared against `minDropoffEarliestTs` and `deliveryByTs` |
| `assignment_id` | Lookup `estimated_delivery_window` |
| `shipment_id` | Check `specialty_client_shipment_ids` |

#### `estimated_delivery_window` fields used

| Field | Role |
|-------|------|
| `assignment_id` | Join key |
| `delivery_by_ts` | `deliveryByTs` for non-specialty shipments |
| `specialty_client_shipment_ids` | Identifies specialty shipments → use `endTs` instead |

## 4. QA / Test notes
<!-- Happy cases, edge cases, sample data, things to watch when testing. -->

Branches to cover when testing (refer to section 3.6):

- **Normal ONTIME:** `min_dropoff_earliest_ts ≤ dropoff_ts ≤ deliveryByTs`.
- **LATE:** `dropoff_ts > deliveryByTs`.
- **EARLY (others):** `dropoff_ts < min_dropoff_earliest_ts`, non-next-day service → denominator only.
- **EARLY → ONTIME:** `COMMINGLE + NEXT_DAY` delivered early.
- **EARLY → LATE:** `ONDEMAND + NEXT_DAY` delivered early (intended semantics, see section 3.6).
- **Boundary:** `dropoff_ts == min_dropoff_earliest_ts` and `dropoff_ts == deliveryByTs` → expect `ONTIME`.
- **Gate skip:** `deliveryByTs = 0` / `min_dropoff_earliest_ts = null` → record not counted.
- **Specialty vs non-specialty:** shipment in/out of `specialty_client_shipment_ids` yields a different `deliveryByTs`.
- **Buffer:** a driver with `buffer > 0%` (any `isNextDay = true` case — both COMMINGLE + NEXT_DAY and ONDEMAND + NEXT_DAY) pushes `endTs` up.
- **Insufficient data:** total records < `min` → negative OTD; `denominator = 0` → insufficient data.

## 5. Related Jira
<!-- Link the epic / tickets for this metric. -->

| Bug / Issue | Impact | Fix |
|-------------|--------|-----|
| `ONDEMAND + NEXT_DAY` early drop was previously counted as `EARLY` instead of `LATE` (OTD inflated) | Resolved | ✅ **Released** — `isNextDay` in `DriverRatingCalculator` now includes `ONDEMAND`; this is the current production logic (see sections 3.2 & 3.6) |
| Specialty client shipment not in `specialty_client_shipment_ids` → uses `delivery_by_ts` (which may be earlier than `max_dropoff_latest_ts`) | Shipment counted as `LATE` when it should be `ONTIME` | Add specialty shipment IDs to `estimated_delivery_window.specialty_client_shipment_ids` via the `specialty_client_ids` Consul key (MOB-2804) |
