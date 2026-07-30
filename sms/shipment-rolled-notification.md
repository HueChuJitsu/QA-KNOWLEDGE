# Recipient SMS — Shipment Rolled Notification (WAT-2163)

**Ticket:** [WAT-2163](https://gojitsu.atlassian.net/browse/WAT-2163) · Rollout: [WAT-2229](https://gojitsu.atlassian.net/browse/WAT-2229)
**Status:** Fully rolled out — enabled for all active regions as of 2026-07-27
**Modules/PRs** (base `release/sms-rolled-shipment`): model [#1701](https://github.com/gojitsucom/model/pull/1701), dao [#2082](https://github.com/gojitsucom/dao/pull/2082), worker [#3120](https://github.com/gojitsucom/worker/pull/3120), recipient-api [#373](https://github.com/gojitsucom/recipient-api/pull/373)
**Related:** [SMS queue logic](sms-queue-logic.md) · [Individual SMS opt-in](../webapp/recipient/functions/individual-sms-opt-in-tracking-page.md)

---

## 1. Overview

When an inbound shipment is **rolled** to the next day (dropped from its route, `assignment_id` nulled), the recipient is notified by SMS that delivery may be delayed. This is a **new SMS type**, `INFORM_RECIPIENT_SHIPMENT_ROLLED`, sent through a dedicated pipeline that respects quiet-hours and re-checks the shipment's state before actually sending — at most **1 SMS per roll**.

This SMS type is **not** in the existing immediate-send list nor the existing `ALL_SMS_QUEUE_TYPES` list documented in [sms-queue-logic.md](sms-queue-logic.md) — it uses its own worker path and its own `sms_queue` `category`, described below.

## 2. Pipeline

| # | Service | What happens |
| --- | --- | --- |
| 1 | `WarehouseManager` (dataorch) | Roll → nulls `assignment_id` → emits event `SHIPMENT.PLANNING.move-date` (`evidence.source=INBOUND`) |
| 2 | `ShipmentRolledHandler` (worker-event-processors-shipment) | Consumes event → publishes `SmsQueueMessage(ROLLED_SHIPMENT)` to the `sms_queue` exchange (+ signal to update the tracking page) |
| 3 | `SmsQueueWorker.processRolledShipmentSms` (worker-sms-queue) | Creates a `sms_queue` row: `status=NOT_CONFIRMED`, `category=NOTIFICATION`, `type=INFORM_RECIPIENT_SHIPMENT_ROLLED`, `reason=ROLLED_SHIPMENT`; deduped per shipment |
| 4 | `SMSQueueProcessSchedule` → `RolledShipmentHandler` (worker-process-sms-queue) | Cron per region. For each row (filtered `sent_count<2`, deduped by `shipmentId`), runs the decision logic below |
| 5 | `SMSComposer` → `CommunicationViaSMSWorker` | Composes the body (see §5) and sends via provider |

**Decision logic** in `RolledShipmentHandler.processSendSMS`:

- `shipment.assignmentId != null` → **suppress** (row → `IGNORED`) — the shipment has since been re-assigned, sending would be a premature/false delay notice.
- `isOutsideWindow(...)` → **defer** (stays `NOT_CONFIRMED`, retried on the next cron run).
- Client doesn't exist → **skip**.
- Otherwise → **send** (row → `SENT`, `sent_count++`) + emits event `SHIPMENT.COMMUNICATION.sms-queue`.

## 3. Why `sms_queue` gained a `category` field

`sms_queue` rows now carry a `category`: `CONFIRMABLE` (access-code flow, the existing types in [sms-queue-logic.md](sms-queue-logic.md) §2) vs. `NOTIFICATION` (this rolled-shipment flow).

- Recipient-api's `DeliveryManager` confirm endpoints only read/write `CONFIRMABLE` rows. Without this split, a recipient confirming their access-code SMS could accidentally flip a rolled `NOTIFICATION` row to `CONFIRMED` and silently drop the rolled SMS.
- `NOTIFICATION` rows are only ever terminated by the schedule (`SENT` / `IGNORED`) — never by a recipient action.
- No migration needed: legacy rows without `category` are read as `CONFIRMABLE` (`category not exists OR = CONFIRMABLE`). No new index required.
- Regression verified in QA: status changes on a `CONFIRM_ACCESS_CODE_MDU_NO_INSTRUCTION` row (e.g. → `OPENED`) do not cross-affect an `INFORM_RECIPIENT_SHIPMENT_ROLLED` row for the same shipment, and vice versa.

## 4. Quiet-hours / send window — resolved in layers, not hardcoded

Quiet-hours = anything outside `customer_available_timewindow`. Logic lives in `CustomerAvailableTimeWindow.isOutsideWindow()`:

| Layer | Source |
| --- | --- |
| 1. Per-client | `client_settings.settings.customer_available_timewindow` (Guava cache, 15 min) |
| 2a. Default (via `CommunicationViaSMSWorker` path) | Consul KV `apps.driverappapi.customer_available_timewindow` |
| 2b. Default (via `SMSQueueProcessSchedule` path) | Hardcoded `.orElse("6:00-22:00")` |
| Gate | Feature flag `ENABLE_CUSTOMER_AVAILABLE_TIME_WINDOW`; if off → `isOutsideWindow` returns `false`, i.e. send regardless of time |
| Timezone | `shipment.timezone`; if null → hardcoded `America/Los_Angeles` |

**Where the feature flag lives:** collection `item_metadata` (Mongo), `scope="APP_FEATURES"`, `owner` ∈ `{CL_<id>, region, AP_DEFAULT}`, lowercase key `enable_customer_available_timewindow` inside `values`. Precedence: client > region > `AP_DEFAULT`. On staging, `AP_DEFAULT = true`.

## 5. Cron — per region

Collection `schedule`, e.g. doc `PROCESS_ROLLED_SHIPMENT_SMS_QUEUE_JFK`: cron `0 */30 7-23`, `time_zone` per region, `attributes.type=ROLLED_SHIPMENT`. The row query uses `fromCreatedDate = startTime − 1 day` to make sure it also picks up today's rows.

## 6. SMS content & sender name — scoped to `INFORM_RECIPIENT_SHIPMENT_ROLLED` only

> **Scope note:** the format of an SMS is defined per message `type` in the `telephonies` table (column `sms_macro`) — each SMS type has its own template. This section documents content resolution **only for `INFORM_RECIPIENT_SHIPMENT_ROLLED`**; it is not a general description of every SMS type's macro. For other types, see their own `telephonies` row.

### 6.1 Template row

Table `telephonies` (Postgres), row `id=5017`, `sms_type=INFORM_RECIPIENT_SHIPMENT_ROLLED`, `sms_activated=true`, `client_id=null` (global — no per-client override on staging/prod today). `sms_macro`:

```
Your ${company_name} package is running a little behind schedule and is now expected by tomorrow. It's still in transit, and we'll keep you updated. Please click here for more information: https://gjt.is/{tracking_code}
```

Because this is DB-driven config, text changes to this row don't need a deploy — just wait for the ≤15 min cache.

### 6.2 Placeholder resolution

`SMSComposer` resolves the macro via `AxlStrSubstitutor` (prefix `${`, suffix `}`, resolves through `XPath.xpath(dataMap, key)`). A key missing from the data map is replaced with an **empty string** (not kept as a literal placeholder).

### 6.3 `${company_name}` resolution for this type

For `INFORM_RECIPIENT_SHIPMENT_ROLLED`, `company_name` is resolved by the same generic composer step used to build the send data map, then substituted into the row-5017 macro above. Traced line-by-line in `SMSComposer.java` (repo `worker`, branch `release/sms-rolled-shipment`):

*Step 1 — client-profile cache loader (lines 159–165):*
```java
public SMSComposer withClientProfileDAO(final ClientProfileDAO clientProfileDAO) {
    this.clientProfileDAO = clientProfileDAO;
    clientProfileCache = CacheBuilder.from("maximumSize=1000,expireAfterWrite=10m")
            .build(new CacheLoader<Long, Optional<ClientProfile>>() {
                public Optional<ClientProfile> load(Long id) {
                    return clientProfileDAO.get(id);   // = SELECT * FROM client_profiles WHERE id = :id
                }
            });
}
```
Cache is keyed by `client_profile_id` (not `client_id`), Guava, `maximumSize=1000`, `expireAfterWrite=10m` — a **separate cache** from the 15-min `client_settings` cache used for quiet-hours (§4); don't conflate the two.

*Step 2 — brand override (lines 471–482):*
```java
final String macro = String.valueOf(telephony.smsMacro);
// handle client profile
if (clientProfileCache != null && shipment.clientProfileId != null) {
    Optional<ClientProfile> optClientProfile = clientProfileCache.getUnchecked(shipment.clientProfileId);
    if (optClientProfile.isPresent()) {
        ClientProfile clientProfile = optClientProfile.get();
        if (clientProfile.isUseProfileName) {
            client.company = clientProfile.name;   // override with the brand name
        }
    }
}
```

*Step 3 — into the data map (line 488):*
```java
data.put("company_name", client.company);   // final value used for ${company_name}
```

`substitutor.replace(macro)` then substitutes `${company_name}` from `data.get("company_name")`.

**Resolution formula (for row 5017 / this SMS type):**
```
company_name =
    IF shipment.client_profile_id != null
       AND profile(shipment.client_profile_id).is_use_profile_name == true
    THEN  profile.name                    // brand name from client_profiles
    ELSE  client.company                  // clients.company column (NOT clients.name)
```

Key points:
- Only the exact profile that `shipment.client_profile_id` points to is considered — no lookup via `client_id`, no traversal through `purchaser_id`.
- The deciding flag is `is_use_profile_name` on that same profile (config UI: **Customize Brand > Tracking Page Settings > "Which company name to display?"**).
- If no override applies, it falls back to `clients.company` (the `company` column, not `name`).
- QA (test #23, #24 in §8) confirmed this row reuses the existing composer logic verbatim — no separate/new code path was added for the rolled-shipment macro.

## 7. Unsubscribers interaction (for this notification)

`INFORM_RECIPIENT_SHIPMENT_ROLLED` is classified as a **service/notification** message (`isPromotional = false`), so the standard unsubscribe gate in `TwilioSMSProviderWorker`/`hasUnsubscribed()` applies:

| `type` in `unsubscribers` | Blocks `INFORM_RECIPIENT_SHIPMENT_ROLLED`? |
| --- | --- |
| `SMS_MARKETING` | No — only blocks `isPromotional=true` SMS |
| `SMS_SERVICE` | **Yes** |
| `SMS` (texted STOP) | **Yes** — blocks all SMS |

This is the same suppression mechanism enforced for CTIA/TCPA compliance — see [CTIA message §7](ctia-message.md#7-send-time-gate--exact-implementation) for the full gate logic, `isForceSent` bypass, and phone-format-mismatch caveat. It is not re-documented here; only the interaction is noted.

## 8. Known issues / risks

- `[RISK]` **False `SENT` status.** When the recipient is unsubscribed (`SMS_SERVICE` or `SMS` type) or `shipments.sms_enable:false` blocks the send, the `sms_queue` row is still updated to `SENT` (with `sent_count` incremented) as if delivery succeeded — no SMS actually goes out. Confirmed in QA on both staging and prod. Not yet fixed.
- Opt-in phone is still respected here: an opted-in recipient gets the rolled SMS even if `shipments.sms_enable:false`.
- Known unrelated issue found during this QA cycle: shipment `70633002` (also used in [BUSINESS_CLOSED_ON_DELIVERY_DATE](../business-hour/BUSINESS_CLOSED_ON_DELIVERY_DATE.md) testing) was observed emitting an excessive number of `one-off-failed` events — flagged by Trang Le to Tuyen Bui (2026-07-14); confirmed as a duplicate of a scan-put-away bug in the warehouse app, unrelated to this SMS feature.

## 9. QA verification summary

Verified by Trang Le on staging (2026-07-17) and prod (2026-07-22 – 2026-07-23).

| Area | Result |
| --- | --- |
| Roll inside `customer_available_timewindow` → exactly one delay SMS, same time as tracking page update | Passed (staging + prod) |
| Roll outside window (quiet hours) → no SMS sent, row queued `NOT_CONFIRMED` | Passed (staging + prod) |
| Queued shipment re-assigned before recheck → SMS suppressed, row → `IGNORED` | Passed (staging + prod) |
| Queued shipment still unassigned at recheck → SMS sent | Passed |
| Same message within 24h → deduped, no second SMS | Passed (staging + prod) |
| `shipments.sms_enable:false` → SMS blocked, opt-in-phone recipients still receive it | Passed |
| `category` isolation (`CONFIRMABLE` vs `NOTIFICATION`) → no cross-contamination | Passed |
| Per-client `customer_available_timewindow` + `max_shipment_move_times` honored (client 820 vs 980 tested with different windows) | Passed |
| Sender-name resolution (`is_use_profile_name=true` → brand name) applies unchanged to this SMS type | Passed |
| BETA | Skipped by design — no worker deployed for this flow on BETA |
| `[RISK]` `sms_enable:false` / unsubscribed → row falsely marked `SENT` | Confirmed bug, not fixed |

Opt-in-phone-specific scenarios (customer with opt-in phone, with/without `sms_enable:false`) were skipped in the staging cycle and verified later on prod (shipment `124180653`).

## 10. Rollout (WAT-2229)

**Status:** Done, 2026-07-27 — enabled for all active regions. Enablement was two production **data** changes, no code deploy:

1. Updated the `telephonies` default row (`id=5017`) with the approved copy (the `sms_macro` in §6.1).
2. Inserted 34 new `process_sms_queue` schedule docs into Mongo `schedule` (`attributes.type=ROLLED_SHIPMENT`, cron `0 */30 6-22` region-local) — 28 regions that already had delivery-management schedules, plus 6 active regions that didn't (HOU, MCI, SAT, DTW, BFL, RNO). Combined with the pre-existing `TEST` schedule: 35 regions total.

Regions/timezones covered: `America/Chicago` (DFW, CHI, MKE, AUS, STL, HOU, MCI, SAT), `America/New_York` (ATL, IAD, BWI, JFK, PHL, EWR, ORF, RIC, HPN, HVN, CLE, PIT, CVG, CMH), `America/Detroit` (DTW), `America/Los_Angeles` (SMF, SEA, PDX, SDLAX, LAS, LAX, SFO, BFL, RNO), `America/Phoenix` (PHX), `America/Indiana/Indianapolis` (IND).

**Acceptance criteria:** recipient in an enabled region gets exactly one SMS (working `gjt.is` tracking link) when their shipment rolls and stays unassigned, sent within 6:00–22:30 local; re-assigned before send → no SMS (`IGNORED`); access-code/delivery-management SMS flow unchanged.

## Reference queries

```sql
-- Template row for this SMS type
SELECT * FROM telephonies WHERE id = 5017;
```

```js
// sms_queue rows for a shipment, with category
db.sms_queue.find({ shipment_id: <id> }).sort({ _id: -1 })

// Region cron schedule for the rolled-shipment queue
db.schedule.find({ "attributes.type": "ROLLED_SHIPMENT" })

// Feature flag (quiet-hours gate)
db.item_metadata.find({ scope: "APP_FEATURES", "values.enable_customer_available_timewindow": { $exists: true } })
```

---

> *Compiled from Jira comments on WAT-2163 (staging verification 7/17, prod verification 7/22–7/23) and WAT-2229 (rollout, 7/27), by Trang Le. Not published to Confluence since the Atlassian connector is not authorized in this session — to publish for real, authorize via `/mcp` or claude.ai connector settings and ask again.*
