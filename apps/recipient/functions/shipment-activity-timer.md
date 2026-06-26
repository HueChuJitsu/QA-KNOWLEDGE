# Shipment Activity Timer — Delay / Lost Tracking (WAT-1709)

> QA knowledge doc. Purpose: a single place for QA to look up the full logic, configuration, test cases, edge cases and known issues of this feature.

| | |
|---|---|
| **Jira** | [WAT-1709](https://gojitsu.atlassian.net/browse/WAT-1709) (feature) · Epic [WAT-1665](https://gojitsu.atlassian.net/browse/WAT-1665) 2026Q1 Tracking Page Improvements |
| **Known bug** | [WAT-2138](https://gojitsu.atlassian.net/browse/WAT-2138) — worker fails with "Socket closed" when the query is slow |
| **Component** | Recipient Experience |
| **Confluence** | ENG / … / Recipient App / Feature document → "Shipment Activity Timer — Delay / Lost Tracking (WAT-1709)" |
| **Status** | Shipped (verified on staging / beta / prod) |

---

## 1. What the feature does

Detects inbound shipments that appear **stuck** (delayed/lost) and surfaces it to two audiences:

- **Recipient:** a new milestone/message appears on the tracking page.
- **Operations:** a daily Slack digest is posted to the regional Ops channel, summarising how many shipments are affected, grouped by warehouse + client.

Three independent timers, evaluated nightly:

| Timer | Event signal | Trigger condition | Default lookback | Tracking page (milestone) | Verbiage |
|---|---|---|---|---|---|
| **Not received** | `INBOUND:NOT_RECEIVED` | Client marked it *shipped* but the warehouse never scanned it | 24h | Label Created | "We are still waiting for your package. We will process your package as soon as it arrives." |
| **No activity** | `INBOUND:NO_ACTIVITY` | Received/scanned at least once, but no further scan, not terminal | 48h (~2 days) | Exception | "Your package may be delayed. We are working to process your package for delivery." |
| **Lost** | `INBOUND:LOST` | No scan activity for a long time, not terminal | 120h (~5 days) | Exception | "Your package does not have any new updates… If you do not receive your package by {due date}, please contact {merchant} for further assistance." |

---

## 2. Enable/disable per client (config)

The feature is **OFF by default** and is enabled **per client** via an outbound setting. **Key present = ON, key absent = OFF** (there is no separate `enabled:true` flag).

Client setting (under `outbound_settings`), key `shipment_activity_tracking`, value is a **JSON string**:

```json
"shipment_activity_tracking": "{\"not_received_lookback_hours\": \"24\",\"no_activity_lookback_hours\": \"48\",\"lost_lookback_hours\": \"120\"}"
```

- Each value tunes the lookback (hours) for one timer, as a **string integer** (e.g. `"24"`).
- Missing a sub-key → falls back to a default (see Defaults & quirks).
- Invalid value (e.g. `"24h"` or malformed JSON) → throws, is caught, logged as an error, and **that client is skipped** for the run (other clients still process).

Related setting that controls the Lost wording:

- `use_merchant_profile_name` = `"true"` → the Lost message shows the sub-brand name (e.g. Flexsport → "Everlane"). Otherwise it shows "the merchant".

---

## 3. End-to-end flow

```
Cron schedule item (23:00 CT daily, 1 item per action)
        │
        ▼
worker-shipment-activity-schedule  (ShipmentActivityScheduleWorker)
        │  1. build time windows from client settings
        │  2. query "stuck" shipments via DAO (24h band)
        │  3a. publish INBOUND signal message per shipment ────────┐
        │  3b. post Slack digest (grouped by region/warehouse/client)
        ▼                                                          ▼
   Ops Slack channel                                     shipment_signal queue
                                                                   │
                                                                   ▼
                                          worker-shipment-signal (ShipmentEventManager)
                                                creates the outbound INBOUND event
                                                                   │
                                                                   ▼
                                          recipient-api (DecorOutboundEvent)
                                            renders the tracking-page message
```

---

## 4. How the worker finds shipments

Class: `ShipmentActivityScheduleWorker` (repo `worker`), deployed as **worker-shipment-activity-schedule**.

### 4.1. Schedule item → action
Three Mongo schedule items fire daily at `0 0 23 * * ?` America/Chicago, each carrying one `action`:
- `not_received_checking` → `markNotReceived()`
- `no_activity_checking` → `markNoActivity()`
- `lost_checking` → `markLost()`

### 4.2. Build time windows — `getScheduleParams()`
1. `scanTime = message.getEndTime()` (firing time, snapped hourly).
2. Load opted-in clients only: `listByExistField("outbound_settings.shipment_activity_tracking")`.
3. Parse the JSON, pick the lookback for the current action.
4. `endTime = scanTime.minusHours(lookback)`.
5. Bucket clients sharing the same `endTime` → one query per window (efficiency).

### 4.3. Query — the **24-HOUR BAND** (very important)

> The query is a **24h band** (`CHECKING_RANGE = 24`), NOT "older than X". For each window: `fromTime = endTime − 24h`, matching a timestamp BETWEEN `fromTime` AND `endTime`. This makes each shipment fall in the band **exactly once** — on the day it crosses the threshold — so it is never re-flagged.

**Not received** — `listShipmentsShippedButNotReceived`:
```sql
SELECT s.* FROM shipments s
WHERE s._deleted IS NOT true
  AND s.inbound_shipped_ts IS NOT NULL
  AND s.inbound_shipped_ts BETWEEN :fromTime AND :endTime   -- shipped ~lookback ago
  AND s.inbound_scan_ts IS NULL                              -- never scanned at the WH
  AND s.client_id IN (:clientIds)
```

**No activity / Lost** — `listByInboundStatusInRange` with `dateField = inbound_scan_ts`:
```sql
SELECT * FROM shipments
WHERE _deleted IS NOT true
  AND status NOT IN (:terminalStatuses)                     -- not terminal
  AND (inbound_status NOT IN ('RECEIVED_DAMAGED','RECEIVED_LEAKING')
       OR inbound_status IS NULL)                            -- skip damaged/leaking; NULL-safe
  AND client_id IN (:clientIds)
  AND inbound_scan_ts IS NOT NULL                            -- must have been scanned at least once
  AND inbound_scan_ts BETWEEN :fromTime AND :endTime         -- last scan ~lookback ago
```

**Terminal statuses** (excluded): `CANCELLED_BEFORE_PICKUP, CANCELLED_AFTER_PICKUP, PICKUP_FAILED, DROPOFF_FAILED, RETURN_FAILED, DROPOFF_SUCCEEDED, RETURN_SUCCEEDED, RETURN_DAMAGED`.

### 4.4. Example (config 24/48/120, firing 2026-06-24 23:00 CT)

| Action | fromTime | endTime | Matches shipments whose… |
|---|---|---|---|
| not_received | 06-22 23:00 | 06-23 23:00 | `inbound_shipped_ts` ∈ band AND never scanned |
| no_activity | 06-21 23:00 | 06-22 23:00 | last `inbound_scan_ts` ∈ band |
| lost | 06-18 23:00 | 06-19 23:00 | last `inbound_scan_ts` ∈ band |

### 4.5. Query time range per timer (by config)

General formula (same for all 3 timers — the band width is always fixed 24h, regardless of the lookback value):

```
endTime   = scanTime − lookback_hours       (scanTime = firing time, snapped hourly)
fromTime  = endTime − 24h
range     = [fromTime, endTime]             (BETWEEN is inclusive on both ends)
```

The only differences per timer: which **config key** supplies `lookback_hours`, and which **column** the range is matched against.

| Timer | Config key | Column matched | Extra conditions |
|---|---|---|---|
| not_received | `not_received_lookback_hours` | `inbound_shipped_ts` | `inbound_scan_ts IS NULL` |
| no_activity | `no_activity_lookback_hours` | `inbound_scan_ts` | not terminal, not damaged/leaking |
| lost | `lost_lookback_hours` | `inbound_scan_ts` | not terminal, not damaged/leaking |

**Assume the run fires at `2026-06-24 23:00 CT` for all examples below.**

#### Not received — matches `inbound_shipped_ts` in range

| Config (hours) | fromTime | endTime | Query range (inbound_shipped_ts BETWEEN …) |
|---|---|---|---|
| `12` | 06-23 11:00 | 06-24 11:00 | `[06-23 11:00, 06-24 11:00]` |
| `24` (default) | 06-22 23:00 | 06-23 23:00 | `[06-22 23:00, 06-23 23:00]` |
| `48` | 06-21 23:00 | 06-22 23:00 | `[06-21 23:00, 06-22 23:00]` |

→ With `24`: catches shipments marked shipped **between 24h and 48h ago** that still have no warehouse scan.

#### No activity — matches `inbound_scan_ts` in range

| Config (hours) | fromTime | endTime | Query range (inbound_scan_ts BETWEEN …) |
|---|---|---|---|
| `48` (default) | 06-21 23:00 | 06-22 23:00 | `[06-21 23:00, 06-22 23:00]` |
| `72` | 06-20 23:00 | 06-21 23:00 | `[06-20 23:00, 06-21 23:00]` |

→ With `48`: catches shipments whose **last scan was between 48h and 72h ago** (not terminal, not damaged).

#### Lost — matches `inbound_scan_ts` in range

| Config (hours) | fromTime | endTime | Query range (inbound_scan_ts BETWEEN …) | lost_due_date (last scan + lookback + 48) |
|---|---|---|---|---|
| `120` (default) | 06-18 23:00 | 06-19 23:00 | `[06-18 23:00, 06-19 23:00]` | last scan + 168h (7 days) |
| `96` | 06-19 23:00 | 06-20 23:00 | `[06-19 23:00, 06-20 23:00]` | last scan + 144h (6 days) |

→ With `120`: catches shipments whose **last scan was between 120h and 144h ago** (not terminal, not damaged).

> **Read it as:** a timer with lookback `N` hours catches items whose relevant timestamp falls **between `N` and `N+24` hours before the run** — never older, never newer. The `+24` is the fixed band width, not part of the config.

### 4.6. ⚠️ Important limitation — NO catch-up
Because of the 24h band, a shipment is evaluated only once, on the day it crosses the threshold. If on that day the feature wasn't enabled / the run didn't execute / the lookback was changed later → the shipment is **permanently missed**, with no retroactive sweep.

> Example: a shipment with `inbound_shipped_ts = 06-10 23:00` can only be caught by the **06-11 23:00** run (band 06-10 23:00 → 06-11 23:00). On the 06-24 run it is outside the band → NOT flagged.

---

## 5. Lost due date (the date shown on the tracking page)

For **LOST** only, the worker stamps an extra attribute `lost_due_date`:
```
lostDueDate = inbound_scan_ts + (lostLookbackHours + 48)
// default 120 → last scan + 168h = last scan + 7 days
```
→ produces "*If you do not receive your package by {last scan + 7 days}…*". Rendered in the shipment's timezone (fallback `America/Los_Angeles`).

---

## 6. Slack alert to Ops

`sendAlert()` groups shipments and renders the Handlebars Mongo `content_templates` record `INBOUND_ACTIVITY_NOTIFICATION`, posting to recipient `operations-{{region}}`.
- Grouped by **region → warehouse → client**.
- Warehouse alias comes from the shipment's warehouse; if a shipment has no warehouse → falls back to the region's "Main" warehouse.
- The digest shows: scan date, total shipment count, per-client shipment count with the lookback (in days).

Example line: *"▪︎ 12 Shipments at SF-Main have no activities for several days: • Everlane: 8 shipments (5 days)"*

---

## 7. Tracking-page rendering (recipient-api)

`DecorOutboundEvent.populate()` sets `client_profile_name`:
```java
if (history.settings != null && "true".equals(history.settings.get("use_merchant_profile_name"))) {
    setAttribute("client_profile_name", history.clientProfile.name);  // sub-brand, e.g. Flexsport
} else {
    setAttribute("client_profile_name", "the merchant");
}
```
Templates required: display templates `INBOUND:NOT_RECEIVED`, `INBOUND:NO_ACTIVITY`, `INBOUND:LOST` + Mongo event template `FAILED.INBOUND.LOST`.

---

## 8. Creating test data for QA

### 8.1. Prerequisites (one-time per environment)
If the feature produces nothing at all, check these first (usually already deployed):
- Mongo `content_templates` record `INBOUND_ACTIVITY_NOTIFICATION` (Slack template).
- 3 Mongo `shipment_activity_schedule` items (actions `not_received_checking` / `no_activity_checking` / `lost_checking`).
- Recipient display templates `INBOUND:NOT_RECEIVED`, `INBOUND:NO_ACTIVITY`, `INBOUND:LOST` + Mongo event template `FAILED.INBOUND.LOST`.

### 8.2. Step 1 — Enable the feature for a test client
Add the setting to the client's `outbound_settings` (via the client/admin tool, or directly in the Mongo client settings):
```json
"shipment_activity_tracking": "{\"not_received_lookback_hours\": \"1\",\"no_activity_lookback_hours\": \"1\",\"lost_lookback_hours\": \"1\"}"
```
💡 **Tip:** on staging, use a **small lookback (e.g. `1` hour)** so the qualifying window becomes "anything 1–25h old" — far easier to hit than 24/48/120h.
(Optional) set `use_merchant_profile_name: "true"` on the client to test the sub-brand wording in the Lost message.

### 8.3. Step 2 — Prepare a qualifying shipment (per timer)
Pick or create a shipment for the enabled client, then set its timestamps so they fall inside the band for the run.

**Band reminder:** for lookback `N` hours and run time `scanTime`, qualifying range = `[scanTime − N − 24h, scanTime − N]`.

| Timer | Required column values |
|---|---|
| not_received | `inbound_shipped_ts` = inside band; `inbound_scan_ts` = NULL (status / inbound_status don't matter) |
| no_activity | `inbound_scan_ts` = inside band; `status` NOT terminal; `inbound_status` NOT `RECEIVED_DAMAGED`/`RECEIVED_LEAKING` (or NULL) |
| lost | same as no_activity, but the band uses the lost lookback; `inbound_scan_ts` NOT NULL |

Concrete example (lookback = 1h, run fires/triggered at time `T`): put the timestamp at `T − 2h` (well inside `[T−25h, T−1h]`).
```sql
-- not_received: shipped 2h ago, never scanned
UPDATE shipments SET inbound_shipped_ts = now() - interval '2 hours',
                     inbound_scan_ts = NULL
 WHERE id = <SHIPMENT_ID>;

-- no_activity / lost: last scanned 2h ago, not terminal / not damaged
UPDATE shipments SET inbound_scan_ts = now() - interval '2 hours'
 WHERE id = <SHIPMENT_ID>;
```
⚠️ The run uses `scanTime` (snapped to the hour, America/Chicago). If you trigger manually with a chosen end-time, place the timestamp inside the band **relative to that time**, not wall-clock "now" if they differ.

### 8.4. Step 3 — Trigger the run
- **Wait for the cron:** fires daily at 23:00 America/Chicago.
- **Manual / synthetic-time trigger (staging):** trigger `worker-shipment-activity-schedule` for the desired action with a chosen end-time. (This is what was used during staging verification — see the synthetic-time entries in the `worker-shipment-activity-schedule` Datadog logs on WAT-1709.)

### 8.5. Step 4 — Verify
- **Recipient tracking page:** the new milestone/message appears (Label Created / Exception). For Lost, check the due date and the merchant vs sub-brand wording.
- **Slack:** the Ops digest in `operations-{region}` with the right counts and lookback-in-days.
- **Logs:** `worker-shipment-activity-schedule` logs `"Loaded N shipments. First 20 shipments [...]"` — confirms which shipments matched.

### 8.6. Re-test / reset tips
- A shipment won't be re-flagged if its **last event is already that timer** — to re-test, set a fresh qualifying timestamp and/or make sure the last event differs.
- To make a shipment **stop** qualifying: set `inbound_scan_ts = now()` (no_activity/lost) or add a scan (not_received) → it moves out of the band.
- Remember the **no catch-up** rule (§4.6): if you miss the band, change the timestamp again — a later run will not pick it up retroactively.

---

## 9. Test checklist (QA)

### Happy path
1. Client enabled, shipment marked shipped but not scanned > 24h → tracking page **Label Created / "still waiting…"** + Slack not-received digest.
2. Shipment scanned once then no further scan > 48h (not terminal) → **Exception / "may be delayed"** + Slack "(2 days)".
3. Shipment with no scan > 120h → **Exception / LOST message** with `lost_due_date` + red Slack "(5 days)".
4. Lost message for a Flexsport (sub-brand) client → "please contact {sub-brand name}…"; other clients → "please contact the merchant…".

### Regression (MUST NOT trigger)
5. Shipment `RECEIVED_DAMAGED` / `RECEIVED_LEAKING` → MUST NOT trigger no_activity / lost.
6. Shipment in a terminal status → MUST NOT generate an event.
7. Shipment re-scanned (updates `inbound_scan_ts`) before the next run → MUST NOT be re-flagged, MUST NOT show twice on recipient.
8. Shipment not scanned / not marked shipped → MUST NOT trigger any event.
9. Client with `shipment_activity_tracking` NOT set → MUST NOT be processed.

### Testing notes
- The schedule runs at 23:00 America/Chicago. On staging you can trigger manually / use synthetic time (check the `worker-shipment-activity-schedule` service logs in Datadog).
- Create the Mongo records first (content template, schedule items, display/event templates) + the client setting — otherwise the feature produces nothing.
- Because it's a **24h band**: to test, the timestamp must land inside the band (now − lookback − 24h → now − lookback), not just "older than lookback".

---

## 10. Defaults & quirks

- Default lookback when a sub-key is missing: no_activity → 48, lost → 120.
- **Quirk:** the `not_received` fallback default is wired to the *lost* constant ("120") instead of 24 — only matters if `not_received_lookback_hours` is missing; when the key is present the configured value is used.
- Values must be string integers; invalid → the client is skipped for that run (error logged).
- Re-scanned shipments self-exclude (a new `inbound_scan_ts` moves them out of the band).
- The worker queries the **PRIMARY DB** directly (not a replica).

---

## 11. Known issue

### WAT-2138 — Worker fails with "Socket closed"
- On enable, the worker fails: `PSQLException: An I/O error occurred while sending to the backend → SocketException: Socket closed`.
- **Not a logic bug.** The `shipments` query runs long (suspected missing index → seq scan); the pool config `removeAbandoned: true` has no explicit timeout → default ~60s → the connection is closed mid-query.
- Proposed fix: add an index for the query (+ consider pointing to a replica, review the timeout, trim `SELECT *`).
- Details: [WAT-2138](https://gojitsu.atlassian.net/browse/WAT-2138).

---

## 12. Services & deployment

| Service / artifact | Type | Role |
|---|---|---|
| **worker-shipment-activity-schedule** | Worker (new) | Runs the cron, queries shipments, publishes signals, posts Slack |
| **worker-shipment-signal** | Worker | Consumes signals, creates the outbound INBOUND event |
| **recipient-api** | API | Renders the tracking-page message (merchant vs sub-brand) |
| `model` | Library | New event signals + `InboundStatus.SHIPPED` |
| `dao` | Library | New shipment queries |

**Config/data required before running:** Mongo `content_templates` (`INBOUND_ACTIVITY_NOTIFICATION`); 3 `shipment_activity_schedule` items; display + event templates; client setting `shipment_activity_tracking` (+ optional `use_merchant_profile_name`); the worker's queue (DEVO-1812).

---

## 13. Pull requests

| PR | Repo | Purpose |
|---|---|---|
| [#1508](https://github.com/gojitsucom/model/pull/1508) | model | Event signals NOT_RECEIVED / NO_ACTIVITY / LOST |
| [#1515](https://github.com/gojitsucom/model/pull/1515) | model | Inbound status SHIPPED |
| [#1854](https://github.com/gojitsucom/dao/pull/1854) | dao | New shipment queries |
| [#1859](https://github.com/gojitsucom/dao/pull/1859) | dao | Add excludeInboundStatuses filter |
| [#1860](https://github.com/gojitsucom/dao/pull/1860) | dao | NULL-safe NOT-IN fix |
| [#2712](https://github.com/gojitsucom/worker/pull/2712) | worker | New ShipmentActivityScheduleWorker |
| [#2719](https://github.com/gojitsucom/worker/pull/2719) | worker | Slack alert |
| [#2721](https://github.com/gojitsucom/worker/pull/2721) | worker | Tracking-page data + lost due date |
| [#2723](https://github.com/gojitsucom/worker/pull/2723) | worker | Group Slack by warehouse + alias |
| [#333](https://github.com/gojitsucom/recipient-api/pull/333) | recipient-api | Lost message: merchant vs sub-brand |

---

_Source of truth is the code in the repos above. If the logic changes, update this file and reference the new PR/ticket._
