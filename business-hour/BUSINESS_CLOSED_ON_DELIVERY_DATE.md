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

There's a business-hour guard **before** entering `oneOffSprinkleV2`, and it differs by scan path — this determines whether `BUSINESS_CLOSED_ON_DELIVERY_DATE` can be reproduced at all.

| | Small parcel (`warehouse-api`) | Large (`inbound-api`) |
| --- | --- | --- |
| Guard condition | `if (assignmentId == null && isClosed)` | `if (!isOpened)` — **ignores assignmentId** |
| Redelivery (`assignmentId != null`) + business closed | **Bypasses guard** → enters sprinkle → `checkDate` → **`BUSINESS_CLOSED_ON_DELIVERY_DATE`** | Still blocked (prints business ZPL) |
| No-route (`assignmentId == null`) + business closed | Blocked ("Business closed do not sort") | Blocked |

> **Consequence:** the `BUSINESS_CLOSED_ON_DELIVERY_DATE` reason is essentially **only reproducible via the small-parcel redelivery path** (`assignmentId != null`). On the large path, a closed business is always intercepted at the guard and never reaches one-off.

`client_service.enable_business_hour` must be enabled for the business-hour check to apply at all (independent of `sprinklable`).

---

## 5. Playbook to reproduce

**Setup:**

1. Shipment as a **small-parcel redelivery**: has `assignment_id`, the `deliverShipment` stop = `FAILED`.
2. `shipment_extra.is_business = true`.
3. `deliverable_address.business_hours[<delivery day>].status = "Closed"`, correct DA linkage (`customer_profile.deliverable_address_id` → correct `da_id`).
4. `client_service.enable_business_hour = true`, `sprinklable = true`.
5. Scan on a day the business is closed (make sure it doesn't fall into TOO_EARLY first).

**Do not use:**
- Large scan (guard blocks first, this reason never appears).
- No-route business-closed shipment (blocked with a different message, never enters sprinkle).

**Verify the result:** query `sprinkling_traces` by `shipment_id`, check `failure_reason = "BUSINESS_CLOSED_ON_DELIVERY_DATE"`.

```js
db.sprinkling_traces.find({ shipment_id: <id> }).sort({ _id: -1 })
```

**Example used during investigation:** shipment `70633002` — small redelivery, `is_business=true`, `Tue=Closed` → correctly produced `BUSINESS_CLOSED_ON_DELIVERY_DATE`.

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
- Business-closed guard differs: small only blocks when `assignmentId == null`; large always blocks.
- Config/rule cache (RMapCache) — after editing Mongo/Postgres, may need to wait for TTL or restart the service.

---

> *Compiled from a staging investigation (ALT-1883), referencing current HEAD of `sortation-bizlogic` / `routing-bizlogic`. Not published to Confluence since the Atlassian connector is not authorized in this session — to publish for real, authorize via `/mcp` or claude.ai connector settings and ask again.*
