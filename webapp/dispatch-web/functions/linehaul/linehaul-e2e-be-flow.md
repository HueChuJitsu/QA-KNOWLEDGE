# Function: Linehaul — Backend E2E Flow (mm-service)

> **App:** Dispatch Web · **Part of:** Linehaul · **Status:** Draft
> **Shared context:** see [../../README.md](../../README.md)
>
> **Source:** `mm-service` repo — CQRS + Event Sourcing + Actor model
> **Write store:** KurrentDB (event log, stream `trip-{uuid}`)
> **Read stores:** PostgreSQL (`trips`, `tasks` tables), RocksDB (embedded cache), plus `linehaul_events` (secondary Postgres event store)

## 1. Description
<!-- What this function does, and which actor it is for -->

Linehaul trips in `mm-service` are ingested from **NetSuite** (source of truth for
trip/shipment data) and exposed to ops/dispatch via REST + gRPC. Every change —
whether from NetSuite import or a manual ops action — flows through the same
write pipeline (**TripActor → event → KurrentDB**) and is asynchronously
projected into read models (**Postgres / RocksDB**) that the API actually
queries. This doc covers the full backend round-trip: NetSuite fetch →
geocode → import → event store → projection → API read, plus the manual
status-override path used by ops.

## 2. Business Flow
<!-- Main steps, entry/exit conditions -->

### 2.1 Entry points (2 independent triggers)

| Trigger | How it starts | Notes |
|---|---|---|
| Scheduled polling | `NetSuiteWorker` actor ticks every **10 minutes** (hardcoded, not configurable) | Only runs if `NETSUITE_SCHEDULE_ENABLED=true` (default `false`) |
| Manual trigger | `POST /api/v1/ingestion/netsuite/trigger` (requires OAuth scope `SCOPES_INGEST`) | Runs the **identical** job on-demand, regardless of the schedule flag. If scheduling is disabled, this is the **only** way trips get imported. |

### 2.2 Flow diagram

```
[Scheduled tick (10 min)] OR [POST /ingestion/netsuite/trigger]
        │
        ▼
1. Fetch ALL shipments from NetSuite (RESTlet, OAuth 1.0a)
   - no date filter, no pagination — always fetches the full set
   - one malformed record in the response JSON fails the ENTIRE fetch
        │  (hard fail here → abort, no shipments processed)
        ▼
2. Batch geocode
   - dedupe all stop addresses across ALL fetched shipments
     (key = lowercased/trimmed addr1+city+state+zip — exact string match, no fuzzy match)
   - single POST to internal geocoder (/v1/geocode), cached 24h
   - per-address failure → log warn, keep location WITHOUT coordinates, continue
   - only a hard count-mismatch from the geocoder aborts the whole batch
        │
        ▼
3. Process each shipment independently (1 failure ≠ batch abort), sequentially
   (not parallel — one shipment fully processed before the next starts)
   for each shipment:
     a. trip_id = UUIDv5(NAMESPACE_OID, shipment_id) → same shipment_id always
        maps to the same trip_id (no lookup table needed — see 3.8)
     b. generate task list from shipment.stops: per stop → Travel (if not
        first stop) + Checkin + Pickup and/or Dropoff (per stop.type) +
        Checkout; each task_id = UUIDv5(trip_uuid + stop content + task-type
        suffix) — see 3.8
     c. build ImportTripCmd { code, label, trip_type, status, metadata: the
        FULL raw NetSuite JSON, tasks } → send to TripActor (per-trip actor
        instance, via GetOrCreateTrip)
     d. inside TripActor, always append a `TripImported` audit event first
        (unconditional, no skip), then run 4 idempotent sub-commands in order:
                        CreateTrip (skip if trip already exists)
                        → AddTask per generated task (skip per-task if that
                          deterministic task_id already exists on the trip)
                        → ChangeTripStatus (skip if new status == current)
                        → UpdateTripMetadata (skip only if the entire JSON
                          blob is identical to what's stored — see 3.4)
     e. (best-effort, only if enabled) resolve client_id from parsed customer
        name(s) and warehouse_id per geocoded stop — sent with force:false so
        a manual ops override is never clobbered by a re-import; failure = warn
        only, not counted as failed
        │
        ▼
4. Each successful command → domain event appended to KurrentDB (stream trip-{uuid})
        │  (async, via $ce-trip category subscription)
        ▼
5. TripProjectorActor updates read models: PostgreSQL (trips/tasks) + RocksDB (cache)
        │
        ▼
6. Ops/QA reads via REST (GET /trips/{id}) or gRPC — served from
   RAM → RocksDB → PostgreSQL → KurrentDB replay, in that latency order
```

### 2.3 Manual status override (parallel path, not via NetSuite)

```
PUT /api/v1/trips/{id}/status  { "status": "scheduled" }
        │  requires OAuth write scope
        ▼
ChangeTripStatusCmd (event_source = ManualOverride)  →  same TripActor → event → projection
```

Two independent write sources exist and are distinguished by the event's
`event_source` field: `NetsuiteImport` vs `ManualOverride`.

## 3. Spec / Rules
<!-- Business rules, validation, states -->

### 3.1 Trip status values

| Status | Origin | Meaning |
|---|---|---|
| `Draft` | Internal | Default on trip creation |
| `Scheduled` | Internal | Scheduled |
| `InProgress` | Internal | Auto-set when a task starts |
| `Paused` | Internal | Paused |
| `Completed` | Internal | All tasks must be `Completed` first |
| `Cancelled` | Internal | Terminal — no further transitions |
| `Planning` | NetSuite | — |
| `PoSent` | NetSuite | PO sent |
| `Confirmed` | NetSuite | — |
| `PickingUp` | NetSuite | — |
| `InTransit` | NetSuite | — |
| `Delivered` | NetSuite | — |
| `Unknown` | Fallback | Unparseable status string from NetSuite |

### 3.2 Status transition validation — NOT uniform

- **Validated transitions** (dedicated domain methods): `complete()` requires
  ALL tasks `Completed` first; `cancel()` blocked only if already `Completed`;
  `add_task()` blocked if trip is `Completed`/`Cancelled`.
- **Unvalidated / free transition**: `ChangeTripStatusCmd` (used by both
  NetSuite import and the manual status endpoint) only checks "is the new
  status the same as current?" — **any status can jump to any other status**,
  there is no transition-graph enforcement. This is how NetSuite-only statuses
  (`Planning`, `PoSent`, `Confirmed`, `PickingUp`, `InTransit`, `Delivered`)
  actually get set in practice.

### 3.3 NetSuite string → DB value mapping (⚠️ inconsistency)

Source field: `netsuiteRecordData.shipStatus` (e.g. `"3. Confirmed"`,
`"6. Delivered"`, `"5. In Transit"` — NetSuite always prefixes with a
`"N. "` step number, stripped during parsing).

NetSuite sends raw strings (e.g. `"1. Planning"`, `"PO Sent"`, `"in_progress"`)
→ parsed (case-insensitive, strips leading `"N. "` prefix) into the
`TripStatus` enum → re-serialized as **SCREAMING_SNAKE_CASE** into the event
payload (e.g. `PICKING_UP`) → written **as-is** into `trips.status` (Postgres,
`VARCHAR(20)`).

However, a few internal event types hardcode **lowercase** values directly
into the same column (`TRIP_CREATED` → `"draft"`, `TASK_STARTED` → forces
`"in_progress"`, `TRIP_COMPLETED` → `"completed"`, `TRIP_CANCELLED` →
`"cancelled"`). **The `status` column therefore mixes both cases** depending
on which event last touched it. All internal read-side filters use
`LOWER(status)`, so querying is safe — but a QA/BI query using
`status = 'picking_up'` (no `LOWER()`) will silently miss rows stored as
`'PICKING_UP'`.

### 3.4 Idempotency rules (import re-run safety)

| Entity | Skip condition |
|---|---|
| Trip creation | Skip if `trip.code` already non-empty (trip exists) |
| Task | Skip per-task if deterministic `task_id` already exists |
| Status | Skip if new status == current status |
| Metadata | Skip only if the **entire** metadata JSON blob is byte-identical; any single field change (even a timestamp) triggers an update |

Re-running the same import twice must NOT create duplicates — the second run
should show `*_skipped` counts rise while `*_created`/`*_added` stay flat.

### 3.5 Geocoding rules

- Addresses are deduped **across all shipments in the batch**, by a normalized
  string key (not fuzzy match) — case/whitespace-insensitive only.
- Pre-existing lat/lng on a NetSuite stop is **not** checked/preserved before
  the batch geocode step.
- One address failing to geocode does **not** block that shipment or any
  other — the task is still created, just without coordinates.
- Only a hard mismatch in result count from the geocoder aborts the whole
  import run.
- No rate limiting / retry on the geocoder call (documented as a future
  enhancement, not implemented).

### 3.6 `POST /ingestion/netsuite/trigger` contract

```json
{
  "success": true,
  "message": "...",
  "total": 10,
  "imported": 8,
  "failed": 2,
  "details": {
    "trips_created": 3, "trips_skipped": 5,
    "tasks_added": 12, "tasks_skipped": 20,
    "status_updated": 2, "status_skipped": 6
  }
}
```

| HTTP status | Meaning |
|---|---|
| `200` | Worker ran; per-shipment failures are folded into `failed`/`details` — a `200` does NOT mean zero errors |
| `500` | Hard failure (NetSuite unreachable, fetch/JSON-parse error) — `success:false`, all counts `0`, `details:null`. **Not** the same as "no new shipments" |
| `503` | No `NetSuiteWorker` configured for this deployment |

### 3.7 NetSuite → Trip field mapping

The write side only carries `code` / `label` / `trip_type` / `status` — it also
stuffs the **entire raw NetSuite shipment JSON** into `metadata` verbatim. The
rest of the field mapping happens on the **read side**, when the projector
re-parses `metadata` back into a shipment struct and fills in `TripView`:

| NetSuite field | `TripView` column |
|---|---|
| `shipmentId` | `external_id` |
| `netsuiteRecordData.name` | `code` (truncated to max length) |
| `netsuiteRecordData.lane` | `label` |
| `netsuiteRecordData.linehaulType` | `trip_type` |
| `netsuiteRecordData.nsLocation` | `hub_code` |
| `netsuiteRecordData.shipStatus` | `status` (parsed into `TripStatus` enum — see 3.3 for the exact string→enum→DB rules) |
| `netsuiteRecordData.jitsuCustomer` | `customers` (split into an array) |
| `netsuiteRecordData.jitsuPO` | `purchase_order` |
| `motorFreightCarrierDetails.tmsCarrier.{name,carrierMC,carrierDOT}` | `carrier_name` / `carrier_mc` / `carrier_dot` |
| `motorFreightCarrierDetails.cost` | `carrier_cost` |
| `motorFreightCarrierDetails.driverName` / `driverPhone` | `driver_name` / `driver_phone` — **plain strings**, this does NOT create/link a Driver entity |
| `motorFreightCarrierDetails.fuel` | parsed but **not** projected (known gap) |
| `origin.shipper` / `destination.consignee` | `origin` / `destination` |
| `origin.tracking.pickupAppointment{Date,Time,TimeZone}` | `pickup_ts` |
| `destination.tracking.deliveryAppointment{Date,Time,TimeZone}` | `delivery_ts` |
| `origin.tracking.pickupArrivalActual{Date,Time}` | `actual_pickup_ts` |
| `destination.tracking.deliveryArrivalActual{Date,Time}` | `actual_delivery_ts` |

Per-stop task fields: `addr1/addr2/city/state/zip/locName` (+ geocoded coords)
→ task `location`; `tracking.appointmentDate/Time/TimeZone` → task
`planned_start` (Checkin/Pickup/Dropoff only, not Travel/Checkout);
`stop.type` (`Pickup` / `Delivery` / `Pickup/Dropoff` / anything else)
determines which of Pickup/Dropoff get generated — an **unrecognized
`stop.type` silently yields only Checkin + Checkout**, no error, no Pickup/Dropoff task.

> QA note: `driverName`/`driverPhone` on a trip are free-text from NetSuite,
> not a resolved driver assignment — do not expect them to match a
> `driver_id` from the `/trips/{id}/assign` endpoint; they're two unrelated
> pieces of data that happen to both render as "driver" in the UI.

### 3.8 Deterministic ID generation (UUID v5) — why re-import doesn't duplicate

Both trip and task IDs are generated with `UUIDv5(NAMESPACE_OID, seed)` — same
namespace + same seed bytes always hashes to the same UUID (SHA-1-based, no
randomness). This is what lets the importer decide create-vs-update **without
a NetSuite-id-to-trip-id lookup table**: it just recomputes the ID and asks
the trip actor "does this already exist?".

| ID | Seed | Consequence |
|---|---|---|
| `trip_id` | `shipment_id` alone | Same `shipmentId` from NetSuite → same `trip_id` forever, on every re-import |
| `task_id` | `trip_id` + stop's `locName+addr1+city+stopType` + a task-type suffix (`travel`/`checkin`/`pickup`/`dropoff`/`checkout`) | Same stop content on the same trip → same `task_id` on every re-import (this is *why* re-running import is a no-op for unchanged tasks) |

Important consequences for testing:
- The task seed is **content-based, not position-based** — if NetSuite returns
  the stops in a different order but with the same address/name/type, the
  `task_id`s stay identical (no duplicate tasks created just from reordering).
- If a stop's `locName`/`addr1`/`city`/`stop.type` changes between two
  imports, that stop gets a **brand-new** `task_id` — the old task is **not**
  updated or removed, it just sits there alongside the new one. There is no
  "update a task's content" path in this pipeline, only add-or-skip.
- **Edge case / known gap:** if a single shipment has two stops with
  *identical* `locName+addr1+city+stop.type` (e.g. two distinct visits to the
  same warehouse, same stop type), both stops hash to the **same** `task_id`
  for a given suffix — the second one will be silently **skipped** as
  "already exists" even though it represents a separate real-world stop. No
  existing automated test covers this; worth a manual probe if a NetSuite
  shipment with duplicate-looking stops shows up.
- Driver/Vehicle IDs are **not** part of this UUID v5 scheme at all — they're
  plain `i64` external IDs supplied by ops/dispatch, unrelated to NetSuite
  ingestion.

### 3.9 Task creation logic (per stop)

Tasks are generated by walking `shipment.stops` **in the order NetSuite
returns them** (no re-sorting). For each stop, up to 4 task types can be
created; per-stop generation logic (repeated for every stop in the list):

| Order | Task type | Created when | Notes |
|---|---|---|---|
| 1 | `Travel` | Stop is **not** the first one in the list | `location` = Route from the *previous* stop to this one; `route_polyline` computed best-effort from both stops' coordinates — `None` if either is missing (e.g. geocode failed) |
| 2 | `Checkin` | **Always**, every stop | `location` = Point at this stop; `planned_start` from `tracking.appointmentDate/Time/TimeZone`; `timezone` = `tracking.appointmentTimeZone` if non-empty |
| 3 | `Pickup` | `stop.type` (case-insensitive) is `"Pickup"` or `"Pickup/Dropoff"` | Same location/planned_start/timezone as Checkin |
| 4 | `Dropoff` | `stop.type` (case-insensitive) is `"Delivery"` or `"Pickup/Dropoff"` | Same location/planned_start/timezone as Checkin |
| 5 | `Checkout` | **Always**, every stop | Same `location` as Checkin; no `planned_start` |

Key rules to test against:
- A stop with `stop.type` **not** matching any of the recognized values
  (typo, unexpected NetSuite value, empty string) silently produces **only**
  `Checkin` + `Checkout` — no `Pickup`/`Dropoff`, no error/warning logged. This
  is easy to miss in QA because the shipment still "imports successfully."
- `stop.type = "Pickup/Dropoff"` is the only value that produces **both**
  `Pickup` and `Dropoff` tasks for the same stop.
- `Travel` is only ever inserted *between* consecutive stops — a 1-stop
  shipment has **no** `Travel` task at all (just Checkin/[Pickup/Dropoff]/Checkout).
- `Travel`/`Checkout` never get a `planned_start` — only `Checkin`/`Pickup`/`Dropoff` do.
- Each task's deterministic ID is derived from **this stop's content**, not
  its position — see 3.8 for the reordering/duplicate-stop consequences.

## 4. QA / Test notes
<!-- Happy cases, edge cases, sample data, things to watch when testing -->

- **Idempotency (critical):** trigger import twice in a row with unchanged
  NetSuite data → 2nd response must show `trips_skipped`/`tasks_skipped`
  increase and `trips_created`/`tasks_added` stay at `0`.
- **Partial status change:** change only 1 shipment's status in NetSuite, then
  trigger → expect only `status_updated` to move; everything else skips.
- **Geocode fail, import still succeeds:** feed an unresolvable/malformed
  address on one stop → expect the trip/task to still be created/updated
  normally, just missing coordinates on that stop; other shipments in the
  same batch unaffected. Check via DB (`tasks` table location column), not
  just logs — the geocode failure log line does **not** include the shipment
  `external_id`, only the normalized address key.
- **Whole-batch abort on malformed shipment:** one shipment with invalid JSON
  in the NetSuite response fails the **entire** fetch (no partial processing)
  — confirm the trigger response reflects a hard failure (`500`), not a
  partial `200`.
- **500 vs 200 with total:0 — don't confuse them:** `total:0` + `success:false`
  (500) = hard failure; `total:0` + `success:true` (200) = genuinely nothing
  new from NetSuite. Check both the HTTP status and `success` field.
- **Status case mismatch:** if writing manual DB assertions, always wrap in
  `LOWER(status)` — the column mixes `"draft"`/`"IN_PROGRESS"`-style casing
  depending on which code path last wrote it (see 3.3).
- **No automated coverage exists today for:** a fully malformed shipment
  record hitting the fetch JSON-parse stage (currently causes total fetch
  failure) — good manual edge case to probe periodically.
- **No metrics/dashboard for geocode failures** — only `log::warn` /
  `log::info` lines (`"Failed to geocode location '{key}': {err}"` and the
  batch summary `"Geocoding completed: {n} success, {n} failed"`). There is no
  Prometheus counter or `/health` signal for this.
- **Where to verify data at each layer:**
  - KurrentDB UI (`localhost:2113` or the environment's instance) — stream
    `trip-{uuid}`, source of truth for event order.
  - Postgres — `trips` (`status`, `version`, `last_event_number`),
    `linehaul_events` (`stream_id`, `event_type`, `version`, `data`),
    `subscription_checkpoints` (projector lag).
  - RocksDB — no direct inspection tool; only visible via log lines like
    `"Loaded trip {} from RocksDB (last_event: {})"`.
  - `trips.version` should always match the event count for that trip's
    KurrentDB stream — a mismatch means the projector is lagging or broken.
