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

## 9. QA verification

Verified by Trang Le on staging (2026-07-17) and prod (2026-07-22 – 2026-07-23), reconstructed from Jira comments on WAT-2163.

### 9.1 Top checklist (highlights)

- Roll inside `customer_available_timewindow` → exactly one delay SMS sent, same time as tracking page update.
- Roll outside window (quiet hours) → no SMS sent, `shipment_id` queued as `NOT_CONFIRMED`.
- Queued shipment re-assigned (`assignment_id` present) at re-check → SMS suppressed, row set `IGNORED`.
- Queued shipment still unassigned at re-check → SMS sent.
- Same message within 24h → deduped, row moved `NOT_CONFIRMED` → `IGNORED`, no second SMS.
- `shipments.sms_enable:false` → SMS blocked, but opt-in-phone recipients still receive it.
- `[RISK]` `shipments.sms_enable:false` / unsubscribed recipients → `sms_queue` row is still marked `SENT` (count incremented) even though no SMS was actually sent — known issue, not fixed (see §8).

### 9.2 Full test results

| No. | Scenario / QA Checklist | Test Data | Evidence / Actual Result | Pass? | Note |
| --- | --- | --- | --- | --- | --- |
| **Edge Cases** | | | | | |
| 1 | `shipments.sms_enable:false` on one shipment for a customer with multiple shipments → that shipment's SMS is blocked, others unaffected | Customer with 2 shipments, one `sms_enable:true` / one `false` | SMS blocked as expected | ✅ | Staging |
| 2 | `shipments.sms_enable:false` blocks send, but `sms_queue` row still updates `NOT_CONFIRMED` → `SENT` and `sent_count` 0→1 even though no SMS was sent | Same as above | Row falsely shows `SENT` | ⚠️ | `[RISK]` Known issue — status doesn't reflect actual delivery, needs follow-up |
| 3 | Recipient in `unsubscribers` (`type = SMS_SERVICE`) → no SMS sent | Phone in unsubscribers list | No SMS delivered | ✅ | Staging. `[RISK]` Same known issue: `sms_queue` row still marked `SENT`, count=1, despite no send |
| 4 | Regression: status change (e.g. `OPENED`) on a `CONFIRM_ACCESS_CODE_MDU_NO_INSTRUCTION` row does not cross-affect an `INFORM_RECIPIENT_SHIPMENT_ROLLED` row for the same shipment, and vice versa | Shipment with both `sms_queue` categories present | No cross-contamination between `CONFIRMABLE` and `NOTIFICATION` rows | ✅ | Staging — validates the `category` field guard (§3) |
| **Schedule Trigger** | | | | | |
| 5 | Cron schedule fires as expected | Manual trigger | Ran successfully | ✅ | Staging |
| 6 | Cron schedule fires on its normal recurring schedule | Scheduled run (30-min cron) | Ran successfully | ✅ | Staging |
| **Immediate Send Window** | | | | | |
| 7 | Roll during `customer_available_timewindow` → recipient gets exactly one delay SMS, sent same time as tracking page update | Recipient `+16462713111` | Message received, content verified | ✅ | Staging |
| 8 | Roll during window → recipient gets exactly one delay SMS with tracking page update | Shipment in TEST region | SMS sent as expected | ✅ | Prod |
| 9 | After cron cutoff time (still within `customer_available_timewindow`) → SMS is sent, status updated to `SENT` with `sent_time` set | — | Status/`sent_time` updated correctly | ✅ | Staging |
| 10 | Same as above, verified on prod | — | Status/`sent_time` updated correctly | ✅ | Prod |
| **Quiet-Hour Queuing** | | | | | |
| 11 | Roll during cron-off time OR outside `customer_available_timewindow` → row created as `NOT_CONFIRMED` / `NOTIFICATION` in `sms_queue` | — | Row created as expected | ✅ | Staging |
| 12 | Roll outside client SMS window → no SMS sent, shipment queued `NOT_CONFIRMED` | Shipment `124180673` | Log: `"Shipment 124180673 outside client SMS window. Defer rolled SMS to next run"` | ✅ | Prod |
| **Recheck Suppress** | | | | | |
| 13 | Queued shipment WITH `assignment_id` at 6am recheck → SMS suppressed, row set `IGNORED` | — | Row updated to `IGNORED` | ✅ | Staging |
| 14 | Same, verified on prod | Shipment `124180673` | Log: `"Shipment 124180673 is assigned again. Suppress rolled SMS"` | ✅ | Prod |
| **Dedup / No Duplicate SMS** | | | | | |
| 15 | Same message re-triggered within 24h → status moves `NOT_CONFIRMED` → `IGNORED`, no second SMS sent | — | No duplicate sent | ✅ | Staging |
| 16 | Same, verified on prod | Shipment `124180655` | No duplicate sent | ✅ | Prod |
| **Opt-In Phone Interaction** | | | | | |
| 17 | Customer with opt-in phone → SMS sent to the recipient-provided opt-in number (checked via `phone_opt_in_consent.sms_opt_in:true` per `shipment_id`) | `opt_in_phone` config, client 820 (QA TEST): `13273245660` | Not exercised in this cycle | — | Skipped in staging — planned for beta/prod |
| 18 | Customer with opt-in phone + `sms_enable:false` → SMS still sent to the opt-in number | — | Not exercised in this cycle | — | Skipped in staging — planned for beta/prod |
| 19 | No opt-in phone, `shipments.sms_enable:true` → SMS sent to `customer.info` phone | — | Sent as expected | ✅ | Staging |
| 20 | Customer with opt-in phone + `sms_enable:false` → SMS still sent to opt-in number | Shipment `124180653` | Sent as expected | ✅ | Prod |
| 21 | No opt-in phone, `shipments.sms_enable:true` → SMS sent to `customer.info` phone | — | Sent as expected | ✅ | Prod |
| **Config / Send Window Verification** | | | | | |
| 22 | `client_settings.settings` per client honors its own `customer_available_timewindow` **and** `max_shipment_move_times` | Client `820`: `00:00-20:00`, max moves `2`, opt-in enabled. Client `980`: `04:00-23:59`, max moves `2`, opt-in enabled | Windows respected per client | ✅ | Staging. `max_shipment_move_times` is a related per-client cap tested alongside the send window in this case — its own mechanics are not traced further here; see `client_settings.settings` for the raw config |
| 23 | SMS content reflects `client_profiles.is_use_profile_name:true` → uses `brand_name` instead of `company_name` | Client with `is_use_profile_name:true` | Correct brand name shown in SMS | ✅ | Staging — confirms §6.3 sender-name logic applies unchanged here |
| 24 | `${company_name}` / brand-name resolution reuses existing sender-name logic without code changes | Shipments `124180655`, `124180651` | Dev confirmed no new code path added | ✅ | Prod. Note: "Need check existing logic" — flagged for confirmation, not a new defect |
| **Environment Coverage** | | | | | |
| 25 | BETA verification | — | No worker deployed for this flow on BETA | — | Skipped by design — not applicable to BETA |

**Sources:** Jira comments on WAT-2163 — staging verification (Trang Le, 7/17), prod verification (Trang Le, 7/22–7/23), BETA skip note (Dung Truong, 7/22).

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
