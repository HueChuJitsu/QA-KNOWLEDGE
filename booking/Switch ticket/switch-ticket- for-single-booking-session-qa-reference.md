# Switch Ticket — Feature Logic (QA Reference)

> Source: [Confluence — Switch a booked ticket](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2599256076/Switch+a+booked+ticket)
> Tickets: [MOB-2917](https://gojitsu.atlassian.net/browse/MOB-2917) (FE, this doc) · [MOB-2593](https://gojitsu.atlassian.net/browse/MOB-2593) (FE, original story) · [MOB-2965](https://gojitsu.atlassian.net/browse/MOB-2965) (bug) · [ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761) (BE)

## Purpose

Lets a driver move an already-booked **ticket** to a different available zone **within the same booking session (BS)**, without canceling the original booking. No cancellation means: the switch does **not** count against the daily ticket limit, does **not** hit the On-Time-Cancel metric, and fires its own distinct analytics event instead of a cancel event.

**Scope:** ticket-to-ticket switching **within one booking session only**. Route (direct) bookings are **not** in scope — the route booking widget (`direct_booking_item.dart`) was untouched by this feature and still only offers Unbook.

Currently in **Staging Review** (as of 2026-07-14/15). Feature-flagged dark by default.

---

## End-to-end flow

```
Driver taps "Edit" on a booked ticket   (only shown when enable_switch_ticket = true; false = Unbook only)
        │
        ▼
Any other group available in this BS? ──No──► Switch button disabled ("no tickets to switch")
        │ Yes
        ▼
Switch Instructions popup → navigate to Booking Session screen (switch mode)
        │
        ▼
Driver taps "Book" on a target ticket (same BS, different zone) → Switch Confirmation popup
        │
   ┌────┴────┐
 Cancel     Switch
   │          │
   ▼          ▼
Untouched   PUT /booking/{id}/switch-group/{currentItemId}/to/{targetGroup}
              │
              ▼
         BookingManager.switchGroup (driver-app-api)
           ├─ load session (fail closed → 409 on load failure)
           ├─ daily working-time-limit check (412 if switching would exceed it)
           ├─ gRPC switchGroupWithModel — atomic swap, implemented server-side in driver-pool-api
           └─ on success: process pickup ETA + send confirmation SMS
              │
              ▼
         Booking screen reflects the new ticket + "switch-ticket" analytics event fired (FE, offline-durable)
```

## Decision logic

**Switch target eligibility** (`canSwitchToGroup` in driver-app):
- Target group must be **available** (has capacity).
- Target group must **not** be the source ticket's own group (can't switch into itself).
- Target **can** be a zone/group the driver already holds a *different* ticket in.
- The **reservation/ticket-count limit is ignored** for switch mode — a switch is a trade (release one, acquire one), not a new booking, so at-limit groups stay switchable.
- Target must be in the **same booking session** — there is no cross-session or route switching.

**Backend switch execution** (`BookingManager.switchGroup`), in order:
1. Load the booking session — **fails closed** (`409`) if it can't load. (Prior behavior: logged a warning and silently skipped the limit check — now fixed to fail closed.)
2. Daily working-time-limit check — computes the **net time delta only** (release current group − acquire target group), not the full time as if newly booked. Blocks with `412` if this would push the driver over their daily limit. This check is a **separate constraint** from the ticket-count limit above; a switch can be blocked here even though it's exempt from the ticket-count limit.
3. Atomic swap via gRPC `switchGroupWithModel` — release + claim happen together, so the driver is never at 0 or 2 tickets.
4. On success: pickup window carried over, pickup ETA processed, confirmation SMS sent, Redis "tickets left" updated.

---

## Configurable parameters

| Parameter | Meaning | Default |
| --- | --- | --- |
| `enable_switch_ticket` | Gates the entire feature — Edit button + switch flow | `false` |
| `text_no_ticket_to_switch` | Overridable message when switch mode has no group to switch into | i18n fallback: *"There are no available tickets to switch to in this session."* |
| `DriverLimitedTime` policy (Mongo `driver_limited_time`, by region + driver type) | Daily working-time limit values used by the switch guard | none — **fails open** (no block) if no policy document matches the driver's region/type |

### `enable_switch_ticket` — what QA sees per value

| Value | Booked ticket screen (Total Booked) | Booking Session Detail screen |
| --- | --- | --- |
| `false` (default) | Only **Unbook** button | Only **Unbook** button |
| `true` | **Edit** button (opens Switch/Unbook popup) | **Edit** button (opens Switch/Unbook popup) |

### How to set `enable_switch_ticket` at each level

Resolved via `GET /app/config`, merging 4 levels. **Priority (highest wins, no OR logic): Driver > Warehouse > Region > Global.**

| Level | Resolved from | How to set |
| --- | --- | --- |
| Driver (highest) | the driver's own UID | `PUT /metadata/DR_<driverId>/APP_CONFIG/enable_switch_ticket` |
| Warehouse | driver's **currently active assignment** (`assignments`, `is_active = true AND status <> 'COMPLETED'`) — the route/job warehouse, not a fixed home warehouse. No active assignment → this level never applies. | `PUT /metadata/WH_<warehouseId>/APP_CONFIG/enable_switch_ticket` |
| Region | driver's **active** row in `driver_regions` (`is_active = true`) | `PUT /metadata/RG_<regionCode>/APP_CONFIG/enable_switch_ticket` |
| Global (lowest) | Consul KV | Edit `apps/driverappapi/mobile_app_config/enable_switch_ticket` directly (staging path shown) |

Body for all three `PUT /metadata/...` calls:
```json
{"type": "java.lang.Boolean", "value": "true"}
```
Remove an override (fall back to the next level down): `DELETE /metadata/{uid}/APP_CONFIG/enable_switch_ticket`.

**Example — enable for one test driver only:**
```
PUT /metadata/DR_11159/APP_CONFIG/enable_switch_ticket
Body: {"type": "java.lang.Boolean", "value": "true"}
```

> ⚠️ `text_no_ticket_to_switch` follows the same 4-level mechanism, but the real key is **not** `no_ticket_to_switch` — the getter lives on `TextConfig` (`groupKey = "text"`), so the actual key is group-prefixed: `text_no_ticket_to_switch`. Neither key is named in any Jira ticket — both were found by reading the driver-app source (`text_config.dart`, `root_config.dart`).

---

## Known gaps / open items

- **Edit button styling** — staging QA (2026-07-12) reported the background still not matching the green design in at least one screen; unfixed as of this writing.
- **BE analytics AC unconfirmed** — ALT-1761 asks for a distinct backend-side "switch" event too; `BookingManager.switchGroup` (driver-app-api) emits nothing itself. Only the FE-side `switch-ticket` event (via `EventController`, offline-durable, not Jitsu) is confirmed.
- **`driver-pool-api` unverified** — the actual gRPC swap (`switchGroupWithModel`) is implemented there; that repo isn't accessible to the QA token, so its change was never independently reviewed.
- **Scope reduction** — MOB-2593 originally asked for switching into both ticket sessions and DB sessions; only ticket-session switching shipped.
## Key files

- driver-app: `lib/screens/booking/switch_ticket/*`, `lib/screens/booking/booking_mixin.dart`, `lib/config/src/root_config.dart`, `lib/config/src/text_config.dart`
- driver-app-api: `resource/BookingRS.java`, `manager/BookingManager.java`, `resource/AppInfoRS.java`, `resource/MetadataRS.java`
- dispatch-bizlogic: `booking/DriverLimitedTimeManager.java`
- dao: `mongo/booking/DriverLimitedTimeMongoDAO.java` (`driver_limited_time` collection), `sql/DriverRegionSQL.java`, `sql/AssignmentSQL.java`

PRs: driver-app #2126, #2139 · driver-app-api #1688, #1710, #1725, #1730, #1718 · dispatch-bizlogic #1451, #1463 · driver-pool-api (server-side, not enumerated). Shipped: driver-app-api **1.3.37**, dispatch-bizlogic **1.0.54**.
