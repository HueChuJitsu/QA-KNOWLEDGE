# BUSINESS_CLOSED_ON_DELIVERY_DATE — One-Off Sprinkle Failure Reason

> **Ticket:** `ALT-1883` · **Env:** `staging` · **Example region:** `JFK / client 980` · **Core lib:** `sortation-bizlogic`

**Labels:** `routing` `sortation` `sprinkling` `business-hour` `ALT-1883`

---

## 1. Overview

`BUSINESS_CLOSED_ON_DELIVERY_DATE` is a new reason in the `OneOffFailure` enum, added by **ALT-1883**. It is produced when a shipment enters `oneOffSprinkleV2` → `checkDate`, and `BusinessHourManager.getDeliveryEndTime()` returns **empty** — meaning the business is closed on the shipment's `target_delivery_date`.

Unlike shipments blocked at the guard *before* reaching sprinkle (printed as ZPL "Business closed do not sort"), this reason only shows up when the shipment actually **entered** `doOneOffSprinkleV2` and was rejected at the `checkDate` step. Golden rule: if there's no document in `sprinkling_traces`, the lib never ran and this check never applied.

---

## 2. Data sources

Business open/closed is decided from two sources living in two different stores — always requires a manual join when debugging:

| Field | Store | Table/Collection | Notes |
| --- | --- | --- | --- |
| `is_business` | Postgres | `shipment_extra` | Set by the `ShipmentBusinessHourHandler` worker when processing a DAS event (residential=false), only when business-hour is **enabled** for the region/client. Manually setting `address_type = COMMERCIAL_BUILDING` has **no** effect. |
| `business_hours[<dayOfWeek>].status` | Mongo | `deliverable_address` (via `customer_profile.deliverable_address_id`) | Compared against `OPEN_STATUS = "Open"`. |

If `shipment_extra.is_business = false` → always treated as **open**, business-hours checks are skipped entirely.

**Gotcha:** the two sources (`is_business` in Postgres and `business_hours` in Mongo) can drift out of sync → wrong behavior. When setting up test data, make sure both agree.

---

## 3. Engine

- Lib: `BusinessHourManager` in `routing-bizlogic`.
- Key functions: `getDeliveryEndTime()`, `canDeliverShipmentOnDate()`.
- `getDeliveryEndTime()` empty ⇒ business is closed on the delivery date ⇒ `checkDate` returns `BUSINESS_CLOSED_ON_DELIVERY_DATE`.

Order of checks inside `checkDate` (only reached after gates A–D pass):

1. **TOO_EARLY** — checked first: `start.plusDays(days).isAfter(now) && inboundReceivedTs != null`
2. **BUSINESS_CLOSED_ON_DELIVERY_DATE** — `getDeliveryEndTime()` empty
3. OK → continue to nearest-neighbor → overload check

Since TOO_EARLY is checked first, a shipment that is both too-early and business-closed will return `TOO_EARLY`, not this reason.

---

## 4. Guard differs by scan path (most important point)

There's a business-hour guard **before** entering `oneOffSprinkleV2`, and it differs by scan path — this determines whether the new `BUSINESS_CLOSED_ON_DELIVERY_DATE` reason can actually be produced.

| | Small parcel (`warehouse-api`) | Large (`inbound-api`) |
| --- | --- | --- |
| Guard condition | `if (assignmentId == null && isClosed)` — **pre-existing logic, unchanged by ALT-1883** | `if (!isOpened)` checked before `oneOffSprinkleV2`, regardless of `assignmentId` — also pre-existing, unchanged |
| Sprinkle (`assignmentId != null`, e.g. `DROPOFF_FAILED`) + business closed | Bypasses guard → enters sprinkle → `checkDate` → **new reason `BUSINESS_CLOSED_ON_DELIVERY_DATE`** (only visible in the `one-off-failed` event / `sprinkling_traces`) | Still blocked (`isOpened=false` → prints business ZPL, `oneOffSprinkleV2` never called) |
| No-route (`assignmentId == null`) + business closed | **Blocked by the old guard**, same as before ALT-1883 — shows the pre-existing "Business closed do not sort" indication on the app UI/print, never reaches `oneOffSprinkleV2`, no new reason emitted | Still blocked, same as above |

> **Important distinction (correcting an earlier note in this doc):** the new `BUSINESS_CLOSED_ON_DELIVERY_DATE` reason is a backend/event-level value — it only shows up in the `one-off-failed` event and `sprinkling_traces` log, **never on the app UI**. When Trang Le's staging verification included screenshots/evidence for the no-route case, that evidence reflects the **pre-existing app-level guard/message** ("Business closed do not sort") working correctly — a regression check confirming the old guard logic is unaffected — **not** proof that the new reason code appears on that path or on the UI. Only the sprinkle (`DROPOFF_FAILED`) case actually reaches `checkDate` and produces the new reason.
>
> **Consequence:** `BUSINESS_CLOSED_ON_DELIVERY_DATE` (the new event/log reason) is only reproducible via **small-parcel sprinkle**. The no-route case still hits the old pre-sprinkle guard on both small and large paths — its "closed" behavior is unchanged, just re-verified as a regression check, not a new-reason scenario.

`client_service.enable_business_hour` must be enabled for the business-hour check to apply at all (independent of `sprinklable`).

---

## 5. Playbook to reproduce

**Setup:**

1. Shipment as a **small-parcel sprinkle**: has `assignment_id`, the `deliverShipment` stop = `FAILED`.
2. `shipment_extra.is_business = true`.
3. `deliverable_address.business_hours[<delivery day>].status = "Closed"`, correct DA linkage (`customer_profile.deliverable_address_id` → correct `da_id`).
4. `client_service.enable_business_hour = true`, `sprinklable = true`.
5. Scan on a day the business is closed (make sure it doesn't fall into TOO_EARLY first).

**Do not use:**
- Large scan — the `isOpened` guard blocks before `oneOffSprinkleV2` regardless of route state, so this reason never appears there.
- No-route small-parcel shipment — this still hits the **old**, pre-existing guard (`assignmentId == null && isClosed`) and never reaches `oneOffSprinkleV2`. It correctly shows the old "Business closed do not sort" behavior on the app, but that is **not** the new reason code — don't use it to try to reproduce/verify `BUSINESS_CLOSED_ON_DELIVERY_DATE`.

**Verify the result:** the new reason only appears in the backend event/log, not on the UI — query `sprinkling_traces` by `shipment_id`, check `failure_reason = "BUSINESS_CLOSED_ON_DELIVERY_DATE"`.

```js
db.sprinkling_traces.find({ shipment_id: <id> }).sort({ _id: -1 })
```

**Example used during investigation:** shipment `70633002` — small sprinkle, `is_business=true`, `Tue=Closed` → correctly produced `BUSINESS_CLOSED_ON_DELIVERY_DATE`.

**Known issue (not a regression, tracked separately):** during staging verification, shipment `70633002` was observed emitting an excessive number of `one-off-failed` events. Trang Le flagged this to Tuyen Bui (2026-07-14); follow-up update confirmed it's a **duplicate issue related to the scan put-away feature in the warehouse app**, unrelated to the `BUSINESS_CLOSED_ON_DELIVERY_DATE` change itself.

---

## 6. Reference queries

### Postgres

```sql
-- Business flag
SELECT * FROM shipment_extra WHERE id = :id;

-- Client service (enable_business_hour)
SELECT sprinklable, enable_business_hour FROM client_service
WHERE client_id = :clientId AND region = :region AND _deleted IS NOT TRUE;

-- Upsert is_business for testing
INSERT INTO shipment_extra (id, is_business, routing_earliest_dropoff_ts, routing_latest_dropoff_ts)
VALUES (:id, true, :earliest_ts, :latest_ts)
ON CONFLICT (id) DO UPDATE SET is_business = true, _updated = NOW();
```

### Mongo

```js
// Business hours of the DA
db.customer_profile.find({ id: "<cp_id>" }, { deliverable_address_id: 1 })
db.deliverable_address.find({ id: "<da_id>" }, { business: 1, business_hours: 1 })

// Sprinkle result trace
db.sprinkling_traces.find({ shipment_id: <id> }).sort({ _id: -1 })
```

---

## 7. Related gotchas

- `is_business` is **not** set from `address_type` — only set by the worker when processing a DAS event.
- The two sources (`is_business` in Postgres vs `business_hours` in Mongo) can drift out of sync → incorrect behavior; check both when debugging.
- Business-closed guard differs: small only blocks pre-sprinkle when `assignmentId == null` (no-route); large always blocks pre-sprinkle via the `isOpened` check regardless of `assignmentId`. Both guards are pre-existing logic, unchanged by ALT-1883.
- The new `BUSINESS_CLOSED_ON_DELIVERY_DATE` reason is **event/log-only** (`one-off-failed` event, `sprinkling_traces`) — it is never surfaced on the app UI. Don't treat a UI screenshot showing a "business closed" message as evidence of the new reason; that UI message comes from the old pre-sprinkle guard, not from `checkDate`.
- The `com.jitsu.manager` fork of `SprinklingManager` inside `inbound-api-grpc` is a **separate copy** and does **not** pick up this change automatically — a version bump of the `com.axlehire` lib does not touch that fork. Whether the fork needs the same fix was an open decision on the ticket.
- Config/rule cache (RMapCache) — after editing Mongo/Postgres, may need to wait for TTL or restart the service.

---

## 8. Test cases (QA verification history)

Source: Trang Le's comments on `ALT-1883`. All environments below verified on the **small-parcel path only** — large path is explicitly out of scope (see §4).

### Staging — 2026-07-14 (passed)

| Scenario | Test data | Expected | Result |
| --- | --- | --- | --- |
| Business-closed failure (new reason) | Small parcel, sprinkle (`DROPOFF_FAILED`), delivery location closed on target date | `one-off-failed` event, `reason = BUSINESS_CLOSED_ON_DELIVERY_DATE` (event/log only) | Passed |
| Business-closed regression (old guard) | Small parcel, no-route, delivery location closed on target date | Pre-existing pre-sprinkle guard still blocks correctly, shows old "Business closed do not sort" UI behavior — **not** the new reason, `oneOffSprinkleV2` never called | Passed |
| Happy path | Small parcel, no-route, business open | One-off sprinkle succeeds unaffected | Passed |
| Happy path | Small parcel, sprinkle (`DROPOFF_FAILED`), business open | One-off sprinkle succeeds unaffected | Passed |
| Too-early regression | Small parcel, sprinkle (`DROPOFF_FAILED`), before sort window opens | `reason = TOO_EARLY` (unchanged, not misclassified as business-closed) | Passed |
| Too-early regression | Small parcel, no-route, before sort window opens | `reason = TOO_EARLY` (unchanged) | Passed |
| Event shape | `one-off-failed` event payload | Event structure/fields unaffected by the new reason | Passed |

Known issue found during this pass: shipment `70633002` emitted an excessive number of `one-off-failed` events — filed separately, confirmed as a duplicate of a scan put-away bug in the warehouse app (not related to this change).

---

> *Compiled from a staging investigation (ALT-1883) and Trang Le's QA verification comments on the ticket, referencing current HEAD of `sortation-bizlogic` / `routing-bizlogic`. Not published to Confluence since the Atlassian connector is not authorized in this session — to publish for real, authorize via `/mcp` or claude.ai connector settings and ask again.*
