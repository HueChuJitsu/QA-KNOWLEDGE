# Switch Ticket Feature (Driver App Booking)

> Source: [Confluence — Switch a booked ticket](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2599256076/Switch+a+booked+ticket)
> Tickets: [MOB-2917](https://gojitsu.atlassian.net/browse/MOB-2917) (FE, this doc) · [MOB-2593](https://gojitsu.atlassian.net/browse/MOB-2593) (FE, original story) · [MOB-2965](https://gojitsu.atlassian.net/browse/MOB-2965) (bug) · [ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761) (BE)
>
> Screenshots below are captured from the driver-app QA build (iOS Simulator).

## Overview

Switch Ticket lets a driver move an already-booked **ticket** to a different available zone **within the same booking session**, without canceling the original booking. Previously, drivers had to cancel a booked ticket to book a new one once they hit their daily ticket limit — causing unnecessary cancellations (hurting the On Time Cancel metric), reduced coverage, and drivers dropping tickets late in the day to chase newly bonused routes. Switching preserves the existing pickup window, does not count against the daily ticket limit, and is logged as a distinct analytics event, separate from cancellations.

**Scope:** this ships as **ticket-to-ticket switching within the same booking session (BS) only** — the driver switches to another zone/group inside the same BS they already booked in. **Route (direct) bookings are out of scope**: `direct_booking_item.dart` (the route booking widget) was not touched by this feature and still only offers Unbook, no Edit/Switch.

The feature ships behind the remote-config flag `enable_switch_ticket` (default `false`) and is currently in **Staging Review** (as of 2026-07-14). FE work is tracked here (MOB-2917 / MOB-2593); the backend is **[ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761)** (Algorithmic Team), verified passed on staging/beta/prod.

### TL;DR

- **Gate:** the whole feature sits behind `enable_switch_ticket`, resolved per request via `GET /app/config` with precedence **Driver > Warehouse > Region > Global** — the closer level wins outright, no OR logic. See [Feature Flags](#feature-flags).
- **Target selection (FE):** `canSwitchToGroup` only offers other zones **within the same booking session** as the source ticket, excludes the ticket's own group, and ignores the reservation/ticket-count limit entirely. Route bookings are not eligible — Edit/Switch is not wired into the route booking widget at all.
- **Swap (BE):** `PUT /booking/{id}/switch-group/{currentItemId}/to/{targetGroup}` → `BookingManager.switchGroup` loads the session **fail-closed** (409 on load failure), then swaps via gRPC `switchGroupWithModel` — contract + client live in **dispatch-bizlogic**, but the RPC is **implemented server-side in `driver-pool-api`** (repo not accessible to the QA token — never independently verified). The swap is atomic: the original ticket releases only after the new one is confirmed, so the driver is never at 0 or 2 tickets.
- **Guard:** before swapping, a daily working-time-limit check blocks with `412` if the net time delta (release current group − acquire target group) would exceed the driver's daily limit, using a policy provisioned per region (see [Core Rule](#core-rule--switch-api)). This is unrelated to the per-session ticket-count limit, which switches bypass completely.
- **After success:** pickup window carried over, pickup ETA processed + confirmation SMS sent, Redis "tickets left" updated, FE fires the offline-durable `switch-ticket` analytics event.
- **Known gap:** ALT-1761's AC asks for a distinct BE-side event too, but `BookingManager.switchGroup` emits nothing itself — only the FE-side `switch-ticket` event is confirmed today.

---

## How It Works

### Entry Point — Edit Button

On the Total Booked screen and the Booking Session (BS) detail screen, an already-booked ticket shows either **Unbook** or **Edit** depending on `enable_switch_ticket` — see [Feature Flags](#feature-flags) for the exact per-screen behavior.

### Edit Popup

Tapping Edit opens a popup with **Switch** and **Unbook**. It pre-checks whether any other group is available to switch into — spinner while loading, disabled "no available tickets" state if none, so the driver never enters an empty switch screen.


```dart
// lib/screens/booking/switch_ticket/edit_booking_popup.dart
final session = await ref.read(bookingHttpProvider).getSession(...);
canSwitch = visibleGroups(session: session, switchMode: true, switchSourceTicketId: ...).isNotEmpty;
// on error: canSwitch = true — fail open, let the session screen handle the empty case
```

### Switch Instructions Popup

One-time explainer shown after tapping Switch: tells the driver they'll land on the Booking Session screen and tap "Book" on a target ticket to complete the switch.

### Switch Target Selection

While in switch mode, only switchable groups **within the same booking session as the source ticket** are listed — excludes the ticket's own group, includes other zones/groups the driver already holds a *different* ticket in, and **ignores the reservation/ticket-count limit** (a switch is a trade, not a new booking). There is no cross-session or route switching — the target must be another group in the same BS.

```dart
// lib/screens/booking/booking_mixin.dart
bool canSwitchToGroup({required session, required group, required sourceTicketId}) {
  return isAvailable(session: session, group: group) && !isSourceGroup(group: group, sourceTicketId: sourceTicketId);
}
```

### Switch Confirmation Popup

Shows only the destination ticket, plus the pickup-ETA carry-over note (reserved ETA shown as unchanged, or a generic "reserve your ETA" notice if none). This notice is still a static i18n string — an attempt to make it remote-configurable (PR #2145) was closed unmerged.

### Switch Execution & Analytics

On confirmation: calls the switch API, fires the `switch-ticket` analytics event (fire-and-forget, via the offline-capable `EventController` — not Jitsu, so it survives connectivity loss), and refreshes schedules.

```dart
// lib/screens/booking/booking_session/booking_session_detail_notifier.dart
await repository.switchTicket(sessionId: sessionUID, ticketId: ticketId, groupId: groupId);
unawaited(_logSwitchTicketEvent(...));
unawaited(scheduleNotifier.loadSchedules());
```

---

### Feature Flags

#### `enable_switch_ticket`

Gates the Edit/Switch entry point on the **Booked ticket screen** (Total Booked) and the **Booking Session Detail screen** (BS) — both read the flag directly (`DriverAppConfig.instance.root.enableSwitchTicket`) to pick which button to render on an already-booked ticket.

Consul (staging): https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/driverappapi/mobile_app_config/enable_switch_ticket/edit

| Value | What the driver sees |
| --- | --- |
| `false` **(default)** | Both screens show only the **Unbook** button — no Edit, no Switch entry point at all. |
| `true` | Both screens show the **Edit** button instead of Unbook. Tapping it opens the Edit popup (Switch + Unbook actions). |

Not named in any ticket — found directly in `root_config.dart`, `booked_ticket_item_box.dart:78`, and `booked_ticket_item.dart:196`.

#### `text_no_ticket_to_switch`

Consul (staging): https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/driverappapi/mobile_app_config/text_no_ticket_to_switch/edit

Controls the message shown on the **Booking Session screen, in switch mode**, when there is no other group left to switch into:

| State | What the driver sees |
| --- | --- |
| Not set (any level) | Default text: *"There are no available tickets to switch to in this session."* |
| Set | The configured text is shown instead. |

**Not** `no_ticket_to_switch` — the getter lives on `TextConfig`, which has `groupKey = "text"`, so the real override key is group-prefixed (`text.no_ticket_to_switch` → `text_no_ticket_to_switch`). Optional — the feature works fine unset. Not named in any ticket.

```dart
// lib/config/src/root_config.dart
bool get enableSwitchTicket => getBool("enable_switch_ticket", defaultValue: false);
```

**Resolution levels & priority.** Resolved via `GET /app/config`, merging overrides from 4 levels. **Priority (highest wins, no OR logic): Driver > Warehouse > Region > Global.** An override at a higher level completely replaces the lower one's value — `true` or `false`, whichever is set closest to the driver:

```java
// driver-app-api/resource/AppInfoRS.java
List<String> uids = ImmutableList.of(UIDFactory.REGION, UIDFactory.WAREHOUSE, UIDFactory.DRIVER);
for (String uid : uids) {
    metadataList.forEach(metadata -> {
        if (uid.equals(metadata.owner.type)) addMetadataToMap(metadata, conf); // conf.put — later wins
    });
}
```

- **Global** — Consul KV `mobileAppConfig.enable_switch_ticket`.
- **Region** — Mongo `item_metadata` (owner `RG_<regionCode>`, scope `APP_CONFIG`); resolved from the driver's **active** row in `driver_regions` (`is_active IS TRUE`).
- **Warehouse** — same store (owner `WH_<warehouseId>`); resolved from the driver's **currently active assignment** (`assignments`, `is_active IS TRUE AND status <> 'COMPLETED'`) — the current route's warehouse, not a fixed home warehouse. No active assignment → no warehouse-level override applies.
- **Driver** — same store (owner `DR_<driverId>`) — highest precedence.

Overrides are managed via the admin-only `PUT /metadata/{uid}/{scope}/{key}` (`MetadataRS.java`).

**How to set up each level:**

| Level | Where it's resolved from | How to set it |
| --- | --- | --- |
| **4. Global** (lowest priority) | Consul KV `mobileAppConfig.enable_switch_ticket` | Edit the Consul key directly, e.g. staging: `apps/driverappapi/mobile_app_config/enable_switch_ticket` → set value `true`/`false`. Applies to every driver with no other override. |
| **3. Region** | Driver's **active** row in `driver_regions` (`is_active IS TRUE`) | `PUT /metadata/RG_<regionCode>/APP_CONFIG/enable_switch_ticket` with body `{"type": "java.lang.Boolean", "value": "true"}`. Applies to all drivers whose active region matches `<regionCode>`. |
| **2. Warehouse** | Driver's **currently active assignment** (`assignments`, `is_active IS TRUE AND status <> 'COMPLETED'`) — the warehouse of the route/job the driver is on right now, not a fixed home warehouse | `PUT /metadata/WH_<warehouseId>/APP_CONFIG/enable_switch_ticket` with the same body. No active assignment → this level never applies for that driver. |
| **1. Driver** (highest priority) | The driver's own UID | `PUT /metadata/DR_<driverId>/APP_CONFIG/enable_switch_ticket` with the same body. Overrides every other level — use for per-driver testing/rollout. |

All three overrides go through the same admin-only endpoint (`MetadataRS.java`, scope always `APP_CONFIG`):

```
PUT /metadata/{uid}/APP_CONFIG/enable_switch_ticket
Body: {"type": "java.lang.Boolean", "value": "true"}
```

To remove an override and fall back to the next level down: `DELETE /metadata/{uid}/APP_CONFIG/enable_switch_ticket`.

**Example — enabling for one test driver only, id `11159`:**
```
PUT /metadata/DR_11159/APP_CONFIG/enable_switch_ticket
Body: {"type": "java.lang.Boolean", "value": "true"}
```
This turns the feature on for driver 11159 regardless of their region/warehouse/global setting, and does not affect any other driver.

---

### Core Rule — Switch API

```java
// driver-app-api/resource/BookingRS.java
@Path("/{id}/switch-group/{currentItemId}/to/{targetGroup}")
@PUT
public Optional<String> switchGroup(@Auth User user, @PathParam("id") UID bookingId,
    @PathParam("currentItemId") UID ticketUID, @PathParam("targetGroup") UID targetGroupUID) {
  return bookingManager.switchGroup(user, bookingId, ticketUID, targetGroupUID.id);
}
```

`BookingManager.switchGroup`:

1. Loads the booking session — **fails closed** (409) if it can't load (previously logged a warning and silently skipped the limit check instead).
2. Runs the daily working-time-limit check (net delta: release current group, acquire target group) — its 412 propagates outside the fail-closed catch.
3. Calls gRPC `switchGroupWithModel` to swap.
4. On success: processes pickup ETA + sends confirmation SMS for the new ticket.

```java
// dispatch-bizlogic/booking/DriverLimitedTimeManager.java
Double totalTimeAfterSwitch = totalCurrentTime - currentGroupTime + targetGroupTime; // net change only
if (totalTimeAfterSwitch > driverLimitedTime.totalLimitedTimeInSeconds) {
    throw new ClientResponseException(ERROR_TITLE, ..., Response.Status.PRECONDITION_FAILED); // 412
}
```

This is a **separate constraint** from the per-session ticket-count limit, which switches bypass entirely — unlimited switches per session, but a switch can still be blocked by the driver's total daily working-time cap.

**Provisioning.** The limit values come from a `DriverLimitedTime` policy document (Mongo collection `driver_limited_time`), looked up by `region` + `driver_type` (`IC`/`DSP`/`*`): `total_limited_time_in_seconds`, `ticket_limited_time_in_seconds` (map by ticket type), `error_message`. **Fails open if unprovisioned** — no matching document → the check returns immediately with no block, not an error.

---

## E2E Flow

```
Driver taps "Edit" on a booked ticket
        │
        ▼
Any other group available? ──No──► Switch button disabled
        │ Yes
        ▼
Switch Instructions popup → navigate to Booking Session (switch mode)
        │
        ▼
Driver taps "Book" on a target ticket → Switch Confirmation popup
        │
   ┌────┴────┐
 Cancel     Switch
   │          │
   ▼          ▼
Untouched   PUT /booking/{id}/switch-group/{currentItemId}/to/{targetGroup}
              │
              ▼
         BookingManager.switchGroup
           ├─ load session (fail closed → 409)
           ├─ daily time-limit check (412 if over) ──► error, ticket untouched
           ├─ gRPC switchGroupWithModel (swap)
           └─ on success: pickup ETA + confirmation SMS
              │
              ▼
         Booking screen reflects new ticket + "switch-ticket" analytics event fired
```

---

## API Reference

| Method | Path | Role | Notes |
| --- | --- | --- | --- |
| `PUT` | `/booking/{id}/switch-group/{currentItemId}/to/{targetGroup}` | `driver` | Returns the new ticket ID on success. |
| `GET` | `/app/config` | `driver` | Merged config incl. `enable_switch_ticket`, per Driver > Warehouse > Region > Global. |

```json
// Analytics event (EventController/DeliveryEventProcessor pipeline, not Jitsu)
{
  "type": "TICKET", "action": "OUTBOUND", "event": "switch-ticket",
  "fact": {"session_id": "...", "ticket_id": "...", "group_id": "..."},
  "source": "AP_driver-app-v2"
}
```

---

## Open Items

- **Edit button styling gap** — staging QA (2026-07-12) reported the background still not matching the green design in at least one screen; no fix merged yet.
- **Confirmation notice text** — attempted fix (PR #2145) to make it remote-configurable was closed unmerged (2026-07-10).
- **Design sign-off pending** — staging QA (2026-07-10) proposed accepting the current dialogs "as is"; not yet confirmed by design.
- **Scope reduction** — MOB-2593 originally called for switching into both ticket and DB sessions; struck through, only ticket-session switching shipped.
- **BE analytics AC unconfirmed** — `BookingManager.switchGroup` emits no event itself; whether `driver-pool-api` does downstream is unverified.
- **`driver-pool-api` PR unverified** — the actual gRPC swap implementation lives there; QA token has no access to that repo, so it was never independently reviewed.

---

## References

### Jira Tickets

| Ticket | Title | Status |
| --- | --- | --- |
| [MOB-2917](https://gojitsu.atlassian.net/browse/MOB-2917) | [Pending Release] [FE] Introduce "Switch Ticket" feature | Staging Review |
| [MOB-2593](https://gojitsu.atlassian.net/browse/MOB-2593) | [FE] Introduce "Switch Ticket" feature (original FE story) | Done Pending Release |
| [MOB-2965](https://gojitsu.atlassian.net/browse/MOB-2965) | Bug: cannot switch ticket to a different zone from a group already booked | Done Pending Release |
| [MOB-2659](https://gojitsu.atlassian.net/browse/MOB-2659) | [DESIGN] Introduce "Switch Ticket" feature | Done |
| [ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761) | [BE] Introduce "Switch Ticket" feature (backend) | Done |

### Pull Requests

**Frontend (driver-app):** [#2126](https://github.com/gojitsucom/driver-app/pull/2126) core feature · [#2139](https://github.com/gojitsucom/driver-app/pull/2139) source-group fix + button color

**Backend (ALT-1761):**

| PR | Repo | Purpose |
| --- | --- | --- |
| [#1688](https://github.com/gojitsucom/driver-app-api/pull/1688) | driver-app-api | `switch-group` REST endpoint + manager |
| [#1710](https://github.com/gojitsucom/driver-app-api/pull/1710) | driver-app-api | Enforce daily time-limit check before switch |
| [#1725](https://github.com/gojitsucom/driver-app-api/pull/1725) | driver-app-api | Pickup ETA + confirmation SMS after switch |
| [#1730](https://github.com/gojitsucom/driver-app-api/pull/1730) | driver-app-api | Validate path UIDs (400 on bad format) |
| [#1718](https://github.com/gojitsucom/driver-app-api/pull/1718) | driver-app-api | Refactor target-group to TicketBook UID (`TB_<id>`) |
| [#1451](https://github.com/gojitsucom/dispatch-bizlogic/pull/1451) | dispatch-bizlogic | gRPC contract (`Ticket.proto`) + client |
| [#1463](https://github.com/gojitsucom/dispatch-bizlogic/pull/1463) | dispatch-bizlogic | Daily time-limit check (`DriverLimitedTimeManager`) |
| *(not enumerated)* | driver-pool-api | Server-side `switchGroupWithModel` RPC — repo inaccessible to QA token, unverified |

Shipped versions: driver-app-api **1.3.37**, dispatch-bizlogic **1.0.54**.

### Design & Key Files

| Resource | Value |
| --- | --- |
| Figma | [Driver App — Switch Ticket](https://www.figma.com/design/oko9PhlOmF6xNqhRioySxy/Driver-App?m=auto&node-id=23939-146337&t=dNRysOm2jpSVCkWz-1) |
| Consul (staging) | `apps/driverappapi/mobile_app_config/enable_switch_ticket` |
| driver-app | `screens/booking/switch_ticket/*`, `booking_mixin.dart`, `controllers/event_controller.dart`, `config/src/root_config.dart`, `config/src/text_config.dart`, `config/src/config_group.dart` |
| driver-app-api | `resource/BookingRS.java`, `manager/BookingManager.java`, `resource/AppInfoRS.java`, `resource/MetadataRS.java` |
| dispatch-bizlogic | `booking/DriverLimitedTimeManager.java` |
| dao | `mongo/booking/DriverLimitedTimeMongoDAO.java` (`driver_limited_time`), `sql/DriverRegionSQL.java`, `sql/AssignmentSQL.java` |
