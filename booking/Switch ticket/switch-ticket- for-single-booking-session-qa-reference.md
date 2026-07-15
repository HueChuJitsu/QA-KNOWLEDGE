# Switch Ticket Feature (Driver App Booking)

> Source: [Confluence — Switch a booked ticket](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2599256076/Switch+a+booked+ticket)
> Tickets: [MOB-2917](https://gojitsu.atlassian.net/browse/MOB-2917) (FE, this doc) · [MOB-2593](https://gojitsu.atlassian.net/browse/MOB-2593) (FE, original story) · [MOB-2965](https://gojitsu.atlassian.net/browse/MOB-2965) (bug) · [ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761) (BE)
> Figma: [Driver App — Switch Ticket](https://www.figma.com/design/oko9PhlOmF6xNqhRioySxy/Driver-App?m=auto&node-id=23939-146337&t=dNRysOm2jpSVCkWz-1)

## Overview

Switch Ticket lets a driver move an already-booked **ticket** to a different available zone **within the same booking session**, without canceling the original booking. Previously, drivers had to cancel a booked ticket to book a new one once they hit their daily ticket limit — causing unnecessary cancellations (hurting the On Time Cancel metric), reduced coverage, and drivers dropping tickets late in the day to chase newly bonused routes. Switching preserves the existing pickup window, does not count against the daily ticket limit, and is logged as a distinct analytics event, separate from cancellations.

**Scope:** this ships as **ticket-to-ticket switching within the same booking session (BS) only** — the driver switches to another zone/group inside the same BS they already booked in. **Route (direct) bookings are out of scope**: `direct_booking_item.dart` (the route booking widget) was not touched by this feature and still only offers Unbook, no Edit/Switch.

The feature ships behind the remote-config flag `enable_switch_ticket` (default `false`) and is currently in **Staging Review** (as of 2026-07-14). FE work is tracked here (MOB-2917 / MOB-2593); the backend is **[ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761)** (Algorithmic Team), verified passed on staging/beta/prod.

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

## API Reference

| Method | Path | Role | Notes |
| --- | --- | --- | --- |
| `PUT` | `/booking/{id}/switch-group/{currentItemId}/to/{targetGroup}` | `driver` | Returns the new ticket ID on success. |
| `GET` | `/app/config` | `driver` | Merged config incl. `enable_switch_ticket`, per Driver > Warehouse > Region > Global. |

## How It Works

### Entry Point — Edit Button

On the Total Booked screen and the Booking Session (BS) detail screen, an already-booked ticket shows either **Unbook** or **Edit** depending on `enable_switch_ticket` — see [Feature Flags](#feature-flags) for the exact per-screen behavior.

### Edit Popup

Tapping Edit opens a popup with **Switch** and **Unbook**. It pre-checks whether any other group is available to switch into — spinner while loading, disabled "no available tickets" state if none, so the driver never enters an empty switch screen.

### Switch Instructions Popup

One-time explainer shown after tapping Switch: tells the driver they'll land on the Booking Session screen and tap "Book" on a target ticket to complete the switch.

### Switch Target Selection

While in switch mode, only switchable groups **within the same booking session as the source ticket** are listed — excludes the ticket's own group, includes other zones/groups the driver already holds a *different* ticket in, and **ignores the reservation/ticket-count limit** (a switch is a trade, not a new booking). There is no cross-session or route switching — the target must be another group in the same BS.

### Switch Confirmation Popup

Shows only the destination ticket, plus the pickup-ETA carry-over note (reserved ETA shown as unchanged, or a generic "reserve your ETA" notice if none).

### Switch Execution & Analytics

On confirmation: calls the switch API, fires the `switch-ticket` analytics event (fire-and-forget, via the offline-capable `EventController` — not Jitsu, so it survives connectivity loss), and refreshes schedules.

### Feature Flags

#### `enable_switch_ticket`

Gates the Edit/Switch entry point on the **Booked ticket screen** (Total Booked) and the **Booking Session Detail screen** (BS) — both read the flag directly (`DriverAppConfig.instance.root.enableSwitchTicket`) to pick which button to render on an already-booked ticket.

Consul (staging): https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/driverappapi/mobile_app_config/enable_switch_ticket/edit

| Value | What the driver sees |
| --- | --- |
| `false` **(default)** | Both screens show only the **Unbook** button — no Edit, no Switch entry point at all. |
| `true` | Both screens show the **Edit** button instead of Unbook. Tapping it opens the Edit popup (Switch + Unbook actions). |

#### `text_no_ticket_to_switch`

Consul (staging): https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/driverappapi/mobile_app_config/text_no_ticket_to_switch/edit

Controls the message shown on the **Booking Session screen, in switch mode**, when there is no other group left to switch into:

| State | What the driver sees |
| --- | --- |
| Not set (any level) | Default text: *"There are no available tickets to switch to in this session."* |
| Set | The configured text is shown instead. |

**Resolution levels & priority.** Resolved via `GET /app/config`, merging overrides from 4 levels. **Priority (highest wins, no OR logic): Driver > Warehouse > Region > Global.** An override at a higher level completely replaces the lower one's value — `true` or `false`, whichever is set closest to the driver:

- **Global** — Consul KV `mobileAppConfig.enable_switch_ticket`.
- **Region** — Mongo `item_metadata` (owner `RG_<regionCode>`, scope `APP_CONFIG`); resolved from the driver's **active** row in `driver_regions` (`is_active IS TRUE`).
- **Warehouse** — same store (owner `WH_<warehouseId>`); resolved from the driver's **currently active assignment** (`assignments`, `is_active IS TRUE AND status <> 'COMPLETED'`) — the current route's warehouse, not a fixed home warehouse. No active assignment → no warehouse-level override applies.
- **Driver** — same store (owner `DR_<driverId>`) — highest precedence.

Overrides are managed via the admin-only `PUT /metadata/{uid}/{scope}/{key}`

**How to set up each level:**

| Level | Where it's resolved from | How to set it |
| --- | --- | --- |
| **4. Global** (lowest priority) | Consul KV `mobileAppConfig.enable_switch_ticket` | Edit the Consul key directly, e.g. staging: `apps/driverappapi/mobile_app_config/enable_switch_ticket` → set value `true`/`false`. Applies to every driver with no other override. |
| **3. Region** | Driver's **active** row in `driver_regions` (`is_active IS TRUE`) | `PUT /metadata/RG_<regionCode>/APP_CONFIG/enable_switch_ticket` with body `{"type": "java.lang.Boolean", "value": "true"}`. Applies to all drivers whose active region matches `<regionCode>`. |
| **2. Warehouse** | Driver's **currently active assignment** (`assignments`, `is_active IS TRUE AND status <> 'COMPLETED'`) — the warehouse of the route/job the driver is on right now, not a fixed home warehouse | `PUT /metadata/WH_<warehouseId>/APP_CONFIG/enable_switch_ticket` with the same body. No active assignment → this level never applies for that driver. |
| **1. Driver** (highest priority) | The driver's own UID | `PUT /metadata/DR_<driverId>/APP_CONFIG/enable_switch_ticket` with the same body. Overrides every other level — use for per-driver testing/rollout. |

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

## Acceptance Criteria — Switch Ticket Functional (BE)

Covers `BookingManager.switchGroup` / the `switch-group` API (ALT-1761), tested at the API layer. One duplicate was folded before writing these: the generic *"Target session/group unavailable / full / invalid"* case is subsumed by **BE-AC18** (zone at limit) and **BE-AC19** (target group not found), which assert the same outcome with more precise conditions and responses.

**BE-AC1 — Switch to a ticket in a different zone within the same booking session succeeds**
**Given** the driver has an active TICKET-type booking session with 1 booked ticket, and the session has another ticket available in a different zone,
**When** `switch-group` is called with `currentItemId` = the booked ticket and `targetGroup` = a ticket in a different zone,
**Then** the API returns `200` with the new ticket's UID, the driver now holds the new (target) ticket with the original no longer booked, and the ticket limit does not increase (the switch is not counted as a new booking).

**BE-AC2 — Switching a route to another route is not allowed**
**Given** the driver has an active TICKET-type booking session, and both `currentItem` and `targetGroup` are routes (assignments),
**When** `switch-group` is called with both current and target as routes,
**Then** the API returns a "can't switch" error, and the original ticket/assignment remains unchanged (not released).

**BE-AC3 — Switching to a ticket in the SAME zone as the original is not allowed**
**Given** the target ticket belongs to the same zone as the original ticket,
**When** `switch-group` is called with `targetGroup` in the same zone,
**Then** the API returns an error, the change is not allowed, and the driver still holds the original ticket.

**BE-AC4 — Switching to a ticket in a DIFFERENT zone is allowed**
**Given** the target ticket belongs to a different zone than the original, and that zone doesn't match the zone of any other ticket the driver has booked,
**When** `switch-group` is called with `targetGroup` in the different zone,
**Then** the API returns `200` with the new ticket's UID, and the driver holds the new ticket while the original is released.

**BE-AC5 — Switching into a zone matching another already-booked ticket's zone is allowed**
**Given** the driver has previously booked another ticket in zone Z, and the target ticket is in zone Z (different from the original ticket's own zone),
**When** `switch-group` is called with `targetGroup` in zone Z,
**Then** the API returns `200` with the new ticket's UID, and the driver holds the new ticket in zone Z while the original is released.

**BE-AC6 — Switching during an active probation period is allowed**
**Given** the driver is within an active probation period and the target ticket is in a valid zone,
**When** `switch-group` is called during probation,
**Then** the API returns `200` with the new ticket's UID — probation does not block the switch.

**BE-AC7 — A ticket switched into without an ETA is auto-voided after 5 minutes**
**Given** reserve-pickup-ETA is enabled and the original ticket has not booked an ETA,
**When** `switch-group` succeeds and no ETA is reserved for the new ticket within 5 minutes,
**Then** the new ticket is automatically voided after 5 minutes and is no longer booked by the driver.

**BE-AC8 — Switch succeeds when the original zone has no Redis cache, and none is created**
**Given** the original zone has no `NUM_TICKET_LEFT` cache in Redis,
**When** `switch-group` is called successfully,
**Then** the API returns `200` with the new ticket's UID, and the original zone still has no cache afterward — none is auto-created.

**BE-AC9 — Switch updates the original zone's Redis cache**
**Given** the original zone has a `NUM_TICKET_LEFT` cache = *old*,
**When** `switch-group` is called successfully,
**Then** the original ticket is released from that zone and the cache updates to `NUM_TICKET_LEFT_new = old − 1`.

**BE-AC10 — Switch succeeds when the target zone has no Redis cache, and none is created**
**Given** the target zone has no `NUM_TICKET_LEFT` cache in Redis,
**When** `switch-group` is called successfully,
**Then** the API returns `200` with the new ticket's UID, and the target zone still has no cache afterward — none is auto-created.

**BE-AC11 — Switch updates the target zone's Redis cache**
**Given** the target zone has a `NUM_TICKET_LEFT` cache = *old*,
**When** `switch-group` is called successfully,
**Then** the driver takes one ticket in the target zone and the cache updates to `NUM_TICKET_LEFT_new = old + 1`.

**BE-AC12 — Load-detail-booking API reflects updated `group_availability` after switch**
**Given** both the original and target zones have `group_availability` data, recorded before the switch,
**When** `switch-group` is called successfully and the Load detail ticket booking API is called again,
**Then** the original zone's `group_availability` increases by 1 (`old + 1`) and the target zone's decreases by 1 (`old − 1`).

**BE-AC13 — Switch does not count against the ticket limit, even at the max limit**
**Given** the driver currently holds a number of tickets equal to the max daily ticket limit,
**When** `switch-group` is called to another zone,
**Then** the API returns `200` with the new ticket's UID, and the driver's total ticket count remains unchanged (still at the max limit) — the switch is not blocked by the booking limit.

**BE-AC14 — No restriction on the number of times a driver can switch**
**Given** multiple valid zones are available to switch back and forth,
**When** `switch-group` is called repeatedly (e.g. 3–5 times) across different zones,
**Then** every call returns `200` with a new ticket UID, none are blocked by any switch-count limit, and the driver ends up holding the ticket in the last zone switched to.

**BE-AC15 — A switch that would exceed the daily time limit is blocked with 412**
**Given** the target ticket's work duration would push the driver's total daily hours over the daily time limit,
**When** `switch-group` is called to that target,
**Then** the API returns HTTP `412` (PRECONDITION_FAILED) with a message about the time limit, and the driver still holds the original ticket — no switch occurs.

**BE-AC16 — A switch within the daily time limit succeeds**
**Given** the target ticket does not exceed the driver's daily time limit,
**When** `switch-group` is called to that target,
**Then** the API returns `200` with the new ticket's UID and the switch succeeds, not blocked by `412`.

**BE-AC17 — A distinct "Change Ticket" event is emitted, with no cancel event logged**
**Given** event/analytics logging is inspectable,
**When** `switch-group` is called successfully,
**Then** a distinct "Change Ticket" event is recorded, and no cancel/unbook event is emitted for the original ticket.

**BE-AC18 — Switching to a zone at its booking limit (full) is not allowed**
**Given** the target zone has reached its booking limit (`NUM_TICKET_LEFT = 0`),
**When** `switch-group` is called with `targetGroup` in that zone,
**Then** the API returns an error and the switch is not allowed — the driver cannot switch into a full zone.

**BE-AC19 — Switching to a target group that does not exist returns 409**
**Given** the `targetGroup` ID does not correspond to any group in the session,
**When** `switch-group` is called with that non-existent `targetGroup`,
**Then** the API returns HTTP `409 Conflict` with a message such as `"Target group not in session: <id>"`, and the driver's ticket is unchanged.

**BE-AC20 — Switching from a current item that does not exist or is already void fails**
**Given** the `currentItemId` does not exist or refers to an already-voided ticket,
**When** `switch-group` is called with that `currentItemId`,
**Then** the API returns an error, no swap is performed, and the driver's ticket is unchanged.

**BE-AC21 — The 5-minute void timer after a switch counts from the switch time, not the original booking time**
**Given** reserve-pickup-ETA is enabled, the original ticket was booked at T0 without an ETA, and roughly 5 minutes have already elapsed since T0,
**When** `switch-group` is called at T1 (≈ T0 + 4 minutes) to a new ticket without reserving an ETA,
**Then** the switch succeeds; the new ticket is **not** voided at T0 + 5 minutes (the timer does not inherit T0), but **is** auto-voided exactly at T1 + 5 minutes — the countdown restarts from the switch time.

**BE-AC22 — New ticket is not voided past 5 minutes when no valid ETA slot exists to reserve** *(incomplete in source)*
**Given** reserve-pickup-ETA is enabled, the original ticket has no ETA, and no valid ETA time slot is available to reserve,
**When** *(the source test case only specified confirming that no valid ETA slot exists — the switch step and final assertion were not provided)*,
**Then** *(expected result not specified in the source; needs the remaining steps filled in before this can be executed)*.

**BE-AC23 — E2E: Delivery succeeds on a ticket after it has been switched**
**Given** a driver has booked a ticket and then switched to another ticket,
**When** delivery is performed on the ticket after switching,
**Then** the delivery completes successfully.

---

## Acceptance Criteria — Switch Ticket Functional (FE)

### Entry point & button styling

**FE-AC1 — "Book" button changes to "Edit" button on a booked ticket/route**
**Given** the driver is logged in and has successfully booked at least 1 ticket/route,
**When** the driver views the ticket/route card on the Total Booked screen and on the Booking Session (BS) screen containing the same ticket,
**Then** both cards show an "Edit" button instead of "Book", displayed with a filled green background per the final design.

**FE-AC2 — Edit button style on Booked ticket card (Total Booked & BS screen)**
**Given** the driver has at least 1 booked ticket/route,
**When** the driver views the Edit button on the Total Booked screen and on the BS screen showing the same ticket,
**Then** the Edit button has a filled green background on both screens, consistent with the final design.

**FE-AC3 — Detail BS (Reserved ETA): Edit + ETA buttons display correctly**
**Given** the driver has a booked ticket with a Reserved ETA, viewed via the Detail Booking Session screen,
**When** the driver navigates to that screen, taps "ETA", and taps "Edit",
**Then** the screen shows both "Edit" and "ETA" buttons, the ETA popup displays the correct reserved pickup time, and the Edit Ticket popup opens with Switch/Unbook actions.

**FE-AC4 — Detail BS (Unreserved ETA): ETA button on group disabled, Edit enabled**
**Given** the driver has a booked ticket without a Reserved ETA, viewed via the Detail Booking Session screen,
**When** the driver navigates to that screen,
**Then** the ETA button on the group is disabled/greyed out (not tappable), while the Edit button remains enabled and opens the Edit Ticket popup as normal.

### End-to-end switch flow

**FE-AC5 — E2E: Switch ticket from Booked ticket screen (Reserved ETA case)**
**Given** the driver has 1 ticket booked with a Reserved Pickup ETA (e.g. 10:05 AM) and at least 1 other ticket/route is available to switch to,
**When** the driver taps "Edit" on the Total Booked screen → "Switch" → "Got it" on the instructional dialog → "Book" on an available ticket/route → "Switch" to confirm (the confirmation popup summarizes the change and shows the Reserved ETA),
**Then** the switch succeeds, the driver returns to the Active Route/Other tabs with the new ticket reflected as booked, and the old ticket is no longer shown as booked.

**FE-AC6 — E2E: Switch ticket from Detail Booking Session screen**
**Given** the driver has 1 ticket booked, accessible via the Detail Booking Session screen, and at least 1 other ticket/route is available,
**When** the driver taps "Edit" on that screen → "Switch" → "Got it" → "Book" on an available ticket/route → "Switch" to confirm,
**Then** the switch succeeds and the driver returns to the Active Route tab with the new ticket reflected as booked.

**FE-AC7 — E2E: Switch ticket, case without Reserved ETA**
**Given** the driver has 1 ticket booked WITHOUT a Reserved Pickup ETA, and at least 1 other ticket/route is available,
**When** the driver taps "Edit" → "Switch" → "Got it" → "Book" on an available ticket/route → "Switch" to confirm,
**Then** the confirmation popup shown before confirming has no ETA time field, the switch succeeds, the driver returns to the Active Route tab with the new ticket booked, and any "Reserve Pickup ETA by [time]" text still shows if applicable to the new ticket.

**FE-AC8 — E2E: Switch ticket with 2 tickets booked in 2 groups of the same BS**
**Given** the driver has booked 2 tickets, one in each of 2 different groups within the same Booking Session,
**When** the driver taps "Edit" on one of the 2 booked tickets → "Switch" → "Got it" (redirected to Booking Sessions screen showing other available zones, excluding the 2 already booked) → "Book" on an available ticket/route → "Switch" to confirm,
**Then** the switch succeeds for the selected ticket only, and the other booked ticket (in the other group) remains unaffected.

**FE-AC9 — Confirmation popup shows from → to summary**
**Given** the driver is mid-way through the Switch flow and has selected a new ticket,
**When** the driver taps "Book" on the new ticket/route,
**Then** the Switch confirmation popup opens and clearly displays both the current (from) and new (to) ticket info, so the driver can verify the change before confirming.

**FE-AC10 — Tap Cancel on Confirmation popup keeps original ticket unchanged**
**Given** the driver is on the Switch confirmation popup, having selected a new ticket,
**When** the driver taps "Cancel",
**Then** the popup closes, the driver returns to the Booking Session screen with no switch performed, and the original ticket remains booked and unchanged.

**FE-AC11 — Tap-out on Dialog guide to switching dismisses without navigation**
**Given** the driver has tapped "Switch" in the Edit Ticket popup and the instructional dialog is displayed,
**When** the driver taps outside the dialog (tap-out area),
**Then** the dialog is dismissed, the driver remains on the same screen (no navigation to Booking Sessions), and the original ticket remains booked and unaffected.

### Pickup window / ETA carry-over

**FE-AC12 — Deliver_time is carried over correctly after switch (Reserved ETA case)**
**Given** the driver has a booked ticket with a Reserved ETA and a known pickup window (e.g. 9:00–9:30 AM),
**When** the driver performs the Switch flow and views the Switch confirmation popup,
**Then** the popup displays the same Deliver_time/pickup window as the original ticket, and after confirming, the new ticket reflects the same Deliver_time/pickup window carried over.

**FE-AC13 — Deliver_time is carried over correctly after switch (w/o Reserved ETA case)**
**Given** the driver has a booked ticket without a Reserved ETA, with a known pickup window,
**When** the driver performs the Switch flow and views the confirmation popup (w/o Reserved ETA),
**Then** the popup shows the same Deliver_time carried over from the original ticket, and after confirming, the new ticket reflects the same Deliver_time/pickup window as before switching.

**FE-AC14 — ETA time displayed accurately on Confirmation popup (Reserved ETA)**
**Given** the driver has a booked ticket with a specific Reserved Pickup ETA (e.g. 10:05 AM),
**When** the driver starts the Switch flow and selects a new available ticket/route,
**Then** the Switch confirmation popup's ETA time field matches the original ticket's Reserved Pickup ETA exactly.

### Analytics

**FE-AC15 — "Switch Ticket" event is logged distinctly from cancel event**
**Given** analytics/event tracking is available and the driver has a booked ticket eligible to switch,
**When** the driver performs a full successful Switch flow,
**Then** a "Switch Ticket" event is logged with the correct payload (from-ticket ID, to-ticket ID, timestamp), and no "Cancel" event is logged for this action.

**FE-AC16 — Regression: Unbook ticket flow still works correctly and logs the correct event**
**Given** the driver has 1 ticket/route booked and analytics/event tracking is available,
**When** the driver taps "Edit" → "Unbook" → confirms,
**Then** the Edit Ticket popup shows both Switch and Unbook (unaffected by the new feature), the ticket unbooks successfully and reverts "Edit" back to "Book", the existing Unbook/cancel-type event is logged with the correct payload, and no "Switch Ticket" event is logged for this action.

### Availability & edge cases

**FE-AC17 — Switch button disabled when no session is available**
**Given** the driver has 1 booked ticket and no other ticket/route or DB session is currently available to switch to,
**When** the driver taps "Edit",
**Then** the Edit Ticket popup opens with the "Switch" button disabled/greyed out (not tappable), while "Unbook" remains enabled and functional.

**FE-AC18 — Switch button enabled when ≥1 session is available**
**Given** the driver has 1 booked ticket and at least 1 other ticket/route or DB session is available to switch to,
**When** the driver taps "Edit" and then "Switch",
**Then** the "Switch" button is enabled/tappable, and tapping it opens the instructional dialog as expected.

**FE-AC19 — Edit Ticket popup displays correctly when no ticket is available to switch**
**Given** the driver has 1 ticket/route booked and no other ticket/route or DB session is currently available,
**When** the driver taps "Edit" and attempts to tap the disabled "Switch" button,
**Then** the popup opens without error/crash, "Switch" is shown in its disabled/greyed-out style per the final design, tapping it does nothing (the instructional dialog is NOT shown), "Unbook" remains enabled and fully functional, and the overall popup layout/content matches the final design for the no-availability state.

**FE-AC20 — Ticket limit = 1, no ticket booked yet: normal booking confirmation shown**
**Given** the driver's daily ticket limit is 1 and no ticket has been booked yet,
**When** the driver taps "Book" on any available ticket/route and confirms,
**Then** the normal ticket/route booking confirmation popup is shown (not the Switch confirmation popup), the ticket books normally, and its "Book" button changes to "Edit" (per FE-AC1).

**FE-AC21 — Ticket limit = 0 (already booked): Book disabled on other tickets**
**Given** the driver's ticket limit is 1 and the driver has already booked 1 ticket (remaining limit = 0),
**When** the driver views other available tickets/routes and the currently booked ticket,
**Then** the "Book" button on all other tickets/routes is disabled/greyed out, while "Edit" remains available and tappable on the currently booked ticket, allowing Switch or Unbook.

**FE-AC22 — Booking Sessions list: zone fully booked**
**Given** a zone within the Booking Session has reached full booking capacity,
**When** the driver triggers the Switch flow and navigates to the Booking Sessions screen,
**Then** the fully booked zone is shown as not selectable/disabled.

**FE-AC23 — Booking Sessions list: zone reached max bookable count at this time**
**Given** a zone still has unbooked tickets but the driver has reached the max number of tickets bookable at the current time for that zone,
**When** the driver triggers the Switch flow and navigates to the Booking Sessions screen,
**Then** that zone/ticket option is shown as disabled, even though unbooked tickets technically remain.

**FE-AC24 — Booking Sessions list: ≥2 zones available**
**Given** at least 2 zones within the Booking Session have available tickets/routes to switch to,
**When** the driver navigates to the Booking Sessions screen during the Switch flow,
**Then** both zones display as selectable with their available options, and tapping "Book" on an option in either zone independently proceeds to the Switch confirmation popup.

**FE-AC25 — Selected session gets booked by another driver before confirm**
**Given** the driver has selected a new ticket/route and is on the Switch confirmation popup, and another driver books the same ticket/route before this driver confirms,
**When** the driver taps "Switch" to confirm,
**Then** a graceful error message indicates the session is no longer available, the driver is returned to the Booking Sessions screen to pick another option, and the original ticket remains booked and unaffected.

**FE-AC26 — Zone becomes full exactly at the confirm-switch moment**
**Given** the driver has selected a new ticket/route in a zone with exactly 1 remaining slot, and another driver books that last slot right after this driver taps "Book" but before tapping "Switch" to confirm,
**When** the driver taps "Switch" to confirm,
**Then** the confirm fails gracefully with a clear error message (zone now full / session no longer available), the driver returns to the Booking Sessions screen to pick another option, and the original ticket remains booked and unaffected.

### Regression

**FE-AC27 — Driver switches ticket multiple times (≥3) in one session**
**Given** the driver has 1 booked ticket and at least 3 other tickets/routes available to switch to sequentially,
**When** the driver performs Switch #1, immediately performs Switch #2 on the newly booked ticket, and then performs Switch #3,
**Then** each switch succeeds with no restriction or error, none of the switches count against the ticket limit, and the driver still holds exactly 1 booked ticket after all switches.

**FE-AC28 — Unbook flow regression check (Edit Ticket popup)**
**Given** the driver has at least 1 booked ticket/route,
**When** the driver taps "Edit" → "Unbook" → confirms,
**Then** the Edit Ticket popup shows both "Switch" and "Unbook" actions, the unbook dialog is shown (existing flow, unaffected by Switch), and the ticket is unbooked successfully — same behavior as before this feature.

**FE-AC29 — Map button regression check**
**Given** the driver has at least 1 booked ticket/route,
**When** the driver taps "Map" on the Booked ticket screen and on the Detail Booking Session screen,
**Then** the Map screen opens correctly in both cases, unaffected by the new Edit/Switch feature.

### Feature flag on/off/default

**FE-AC30 — Switch Ticket feature is available when `enable_switch_ticket = true`**
**Given** `enable_switch_ticket` is `true` for the driver's app config and the driver has at least 1 ticket/route booked,
**When** the driver navigates to the Total Booked/BS screen, taps "Edit", taps "Switch", and completes the flow,
**Then** the "Edit" button is shown (instead of "Book"), the Edit Ticket popup opens with Switch/Unbook, and the full Switch flow works end-to-end with the switch completing successfully.

**FE-AC31 — Switch Ticket feature is disabled/hidden when `enable_switch_ticket = false`**
**Given** `enable_switch_ticket` is `false` for the driver's app config and the driver has at least 1 ticket/route booked,
**When** the driver navigates to the Total Booked/BS screen and searches the app for any Switch-related UI,
**Then** the ticket/route card shows the original pre-feature UI with no Edit/Switch entry point shown anywhere, the existing cancel/unbook flow still works unaffected, and no Switch-related screen or action (Edit Ticket popup, instructional dialog, Switch confirmation) is reachable anywhere in the app.

**FE-AC32 — Switch Ticket feature behavior when `enable_switch_ticket` uses its default value (unset)**
**Given** `enable_switch_ticket` is not explicitly configured for the driver's app config (falls back to its default, same as `false`) and the driver has at least 1 ticket/route booked,
**When** the driver navigates to the Total Booked/BS screen,
**Then** the app behaves identically to the `false` case — no Edit/Switch entry point is shown, the existing cancel/unbook flow still works, and the app behaves stably and predictably with no crash or inconsistent state due to the missing explicit config.

---

