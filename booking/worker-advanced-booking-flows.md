# Worker Advanced Booking — 2 Flows

## Overview

When worker-advanced-booking is triggered, it performs 2 independent jobs in a single run:

1. **Create Advance Booking Session** — for a future target date
2. **Auto Forecast** — compute `booking_forecast_v2` for the same weekday next week

---

## Flow 1 — Create Advance Booking Session

```
Worker fires TUE 09 Jun (~1am)
        ↓
day_in_advance = 2 → target_date = THU 11 Jun
        ↓
Get ticket_type from booking_session_template
  via refresh_advance_booking_config.steps.template_id = "lax-template"
  → booking_session_template.ticket_type (default = IC if field not present)
        ↓
Read booking_forecast_v2
  week_day = THU, region = LAX, ticket_type = IC
        ↓
Use average_ticket per zone → populate ticket_book
Zone included if empty_allowed=true OR average_ticket > 0
        ↓
Create booking_session
  target_date = THU 11 Jun
  booking_start_time = 17H30M local THU 11 Jun
```

### Key notes

- `day_in_advance` comes from `refresh_advance_booking_config.steps[].day_in_advance`
- `ticket_type` is resolved from `booking_session_template` via `template_id` — defaults to `IC` if not set
- `booking_forecast_v2` is keyed by `week_day` only (no specific date)
- Zone is included if `empty_allowed = true` **OR** `average_ticket > 0`

---

## Flow 2 — Auto Forecast

```
Worker fires TUE 09 Jun (~1am)
        ↓
Calculate for trigger - 1 = MON 08 Jun
(MON booking sessions nearly all complete by ~1am TUE)
        ↓
Query booking_session: region=LAX, ticket_type=IC
  target_date = MON 08 Jun, status ∈ {In_progress, Completed}
        ↓
For each booking_session:
  groups[i].id → query ticket_book by id
    → read ticket_book.attributes:
        tour_cost_min (= minPay), tour_cost_max (= maxPay),
        min_shipment_count, max_shipment_count,
        min_ocac, max_ocac
        ↓
Save as booking_summary
  target_date = MON 08 Jun
        ↓
Look back 3 MON same weekday:
  MON 25 May, MON 01 Jun, MON 08 Jun
  → MAX per field across 3 records (skip missing weeks)
        ↓
Apply ALT-1823 floor per ticket_type:

  | Type  | tour_cost_min | maxPay | minShipment | maxShipment | minOCAC | maxOCAC |
  |-------|--------------|--------|-------------|-------------|---------|---------|
  | IC    | 39           | 105    | 19          | 25          | 0.3     | 4.0     |
  | LARGE | 50           | 220    | 25          | 60          | 1.3     | 5.0     |
  | SUPE  | 39           | 150    | 19          | 25          | 0.3     | 5.0     |

  Rules:
  - calc < floor  → use floor value
  - calc >= floor → keep calculated value
  - no data at all (all 3 weeks missing) → use full defaults from table above
        ↓
Update booking_forecast_v2
  week_day = MON, region = LAX, ticket_type = IC
  (no specific date — overwritten each time worker runs)
        ↓
Save snapshot → booking_forecast_data
  target_date = MON 15 Jun (next MON)
```

### Key notes

- Worker runs at ~1am, so `trigger_date - 1` sessions are nearly all complete
- Lookback window = **3 weeks, same weekday** — hardcoded in BE, not configurable
- `booking_forecast_v2` is keyed by `week_day` only — no specific date
- `booking_forecast_data` is the snapshot with a specific `target_date` — used for audit/tracking
- `tour_cost_min` in collection = `minPay` in UI / ALT-1823 terminology

---

## Relationship Between 2 Flows

```
TUE 09 Jun → worker fires
  ├── Flow 1 → reads booking_forecast_v2 (week_day=THU) → creates BS for THU 11 Jun
  └── Flow 2 → computes booking_summary for MON 08 Jun
               → updates booking_forecast_v2 (week_day=MON)
               → saves booking_forecast_data (target_date=MON 15 Jun)

SAT 13 Jun → worker fires
  └── Flow 1 → reads booking_forecast_v2 (week_day=MON) → creates BS for MON 15 Jun ↑
```

Flow 1 on SAT **depends on** the result of Flow 2 from the previous TUE.

---

## Collections Involved

| Collection | Key | Description |
|---|---|---|
| `refresh_advance_booking_config` | `id` | Config for worker: regions, target_time, steps (day_in_advance, template_id, booking_start_time) |
| `booking_session_template` | `id` | Template config; `ticket_type` field (default IC if absent) |
| `ticket_book` | `id` | Source of truth for forecast attributes (`ticket_book.attributes.tour_cost_min`, etc.) |
| `booking_summary` | `target_date` | Raw historical data per date, derived from `ticket_book.attributes` |
| `booking_forecast_v2` | `week_day` + `region` + `ticket_type` | Forecast values per weekday — overwritten each run; editable on Forecast Management UI |
| `booking_forecast_data` | `target_date` | Snapshot of `booking_forecast_v2` after successful update — audit/tracking only |
| `booking_session` | `id` | Created by Flow 1; `target_date` = trigger + day_in_advance |

---

## Test Data Setup (for ALT-1823 verification)

To test that `tour_cost_min` is correctly floored by ALT-1823:

1. Find `booking_session` records for the target region, ticket_type, and the 3 same-weekday dates in the lookback window
2. Get `groups[i].id` for the zone under test (e.g. `z2`)
3. Query `ticket_book` by that id — set `attributes.tour_cost_min` to a value **< floor** (e.g. `30` for IC/SUPE, `40` for LARGE)
4. Trigger worker — verify `booking_forecast_v2` gets the floor value, not the raw value

> ⚠️ If updating MongoDB directly (not via worker), clear relevant Redis caches after modification.
