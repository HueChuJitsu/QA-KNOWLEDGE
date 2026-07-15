# Switch Ticket Feature (Driver App Booking)

> Source: [Confluence — Switch a booked ticket](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2599256076/Switch+a+booked+ticket)
> Tickets: [MOB-2917](https://gojitsu.atlassian.net/browse/MOB-2917) (FE, this doc) · [MOB-2593](https://gojitsu.atlassian.net/browse/MOB-2593) (FE, original story) · [MOB-2965](https://gojitsu.atlassian.net/browse/MOB-2965) (bug) · [ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761) (BE)
>  Figma | [Driver App — Switch Ticket](https://www.figma.com/design/oko9PhlOmF6xNqhRioySxy/Driver-App?m=auto&node-id=23939-146337&t=dNRysOm2jpSVCkWz-1) |

## Overview

Switch Ticket lets a driver move an already-booked **ticket** to a different available zone **within the same booking session**, without canceling the original booking. Previously, drivers had to cancel a booked ticket to book a new one once they hit their daily ticket limit — causing unnecessary cancellations (hurting the On Time Cancel metric), reduced coverage, and drivers dropping tickets late in the day to chase newly bonused routes. Switching preserves the existing pickup window, does not count against the daily ticket limit, and is logged as a distinct analytics event, separate from cancellations.

**Scope:** this ships as **ticket-to-ticket switching within the same booking session (BS) only** — the driver switches to another zone/group inside the same BS they already booked in. **Route (direct) bookings are out of scope**: `direct_booking_item.dart` (the route booking widget) was not touched by this feature and still only offers Unbook, no Edit/Switch.

The feature ships behind the remote-config flag `enable_switch_ticket` (default `false`) and is currently in **Staging Review** (as of 2026-07-14). FE work is tracked here (MOB-2917 / MOB-2593); the backend is **[ALT-1761](https://gojitsu.atlassian.net/browse/ALT-1761)** (Algorithmic Team), verified passed on staging/beta/prod.

### TL;DR

- **Gate:** the whole feature sits behind `enable_switch_ticket`, resolved per request via `GET /app/config` with precedence **Driver > Warehouse > Region > Global** — the closer level wins outright, no OR logic. See [Feature Flags](#feature-flags).
- **Target selection (FE):** `canSwitchToGroup` only offers other zones **within the same booking session** as the source ticket, excludes the ticket's own group, and ignores the reservation/ticket-count limit entirely. Route bookings are not eligible — Edit/Switch is not wired into the route booking widget at all.
- **Swap (BE):** `PUT /booking/{id}/switch-group/{currentItemId}/to/{targetGroup}` → `BookingManager.switchGroup` loads the session **fail-closed** (409 on load failure), then swaps via gRPC `switchGroupWithModel` — contract + client live in **dispatch-bizlogic**, but the RPC is **implemented server-side in `driver-pool-api`** (repo not accessible to the QA token — never independently verified). The swap is atomic: the original ticket releases only after the new one is confirmed, so the driver is never at 0 or 2 tickets.
- **Guard:** before swapping, a daily working-time-limit check blocks with `412` if the net time delta (release current group − acquire target group) would exceed the driver's daily limit, using a policy provisioned per region (see [Core Rule](#core-rule--switch-api)). This is unrelated to the per-session ticket-count limit, which switches bypass completely.
- **After success:** pickup window carried over, pickup ETA processed, Redis "tickets left" updated, FE fires the offline-durable `switch-ticket` analytics event.
- **Known gap:** ALT-1761's AC asks for a distinct BE-side event too, but `BookingManager.switchGroup` emits nothing itself — only the FE-side `switch-ticket` event is confirmed today.

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

Not named in any ticket — found directly in `root_config.dart`, `booked_ticket_item_box.dart:78`, and `booked_ticket_item.dart:196`.

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

Overrides are managed via the admin-only `PUT /metadata/{uid}/{scope}/{key}` (`MetadataRS.java`).

**How to set up each level:**

| Level | Where it's resolved from | How to set it |
| --- | --- | --- |
| **4. Global** (lowest priority) | Consul KV `mobileAppConfig.enable_switch_ticket` | Edit the Consul key directly, e.g. staging: `apps/driverappapi/mobile_app_config/enable_switch_ticket` → set value `true`/`false`. Applies to every driver with no other override. |
| **3. Region** | Driver's **active** row in `driver_regions` (`is_active IS TRUE`) | `PUT /metadata/RG_<regionCode>/APP_CONFIG/enable_switch_ticket` with body `{"type": "java.lang.Boolean", "value": "true"}`. Applies to all drivers whose active region matches `<regionCode>`. |
| **2. Warehouse** | Driver's **currently active assignment** (`assignments`, `is_active IS TRUE AND status <> 'COMPLETED'`) — the warehouse of the route/job the driver is on right now, not a fixed home warehouse | `PUT /metadata/WH_<warehouseId>/APP_CONFIG/enable_switch_ticket` with the same body. No active assignment → this level never applies for that driver. |
| **1. Driver** (highest priority) | The driver's own UID | `PUT /metadata/DR_<driverId>/APP_CONFIG/enable_switch_ticket` with the same body. Overrides every other level — use for per-driver testing/rollout. |

```
PUT /metadata/{uid}/APP_CONFIG/enable_switch_ticket
Body: {"type": "java.lang.String", "value": "true"}
```

To remove an override and fall back to the next level down: `DELETE /metadata/{uid}/APP_CONFIG/enable_switch_ticket`.

**Example — enabling for one test driver only, id `11159`:**
```
PUT /metadata/DR_11159/APP_CONFIG/enable_switch_ticket
Body: {"type": "java.lang.String", "value": "true"}
```
This turns the feature on for driver 11159 regardless of their region/warehouse/global setting, and does not affect any other driver.

## Acceptance Criteria — `enable_switch_ticket` Config Precedence

Covers the **Driver > Warehouse > Region > Global** precedence end-to-end. All cases assume the driver's active region/warehouse/driver-ID are already known (resolve via the "How to set up each level" table above if needed).

**AC1 — Resolves from REGION when WAREHOUSE and DRIVER are unset**
**Given** `enable_switch_ticket` is configured only at REGION level (`= true`), with WAREHOUSE and DRIVER unset for this driver,
**When** a driver belonging to that region (no warehouse/driver override) logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to `true` (from REGION), and the "Edit" button and Switch flow are available.

**AC2 — Resolves from WAREHOUSE when only WAREHOUSE is configured**
**Given** `enable_switch_ticket` is configured only at WAREHOUSE level (`= true`), with REGION and DRIVER unset,
**When** a driver belonging to that warehouse logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to `true` (from WAREHOUSE), and the "Edit" button and Switch flow are available.

**AC3 — Resolves from DRIVER when only DRIVER is configured**
**Given** `enable_switch_ticket` is configured only at DRIVER level (`= true`) for a specific driver, with REGION and WAREHOUSE unset,
**When** that driver logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to `true` (from DRIVER), and the "Edit" button and Switch flow are available for that driver only.

**AC4 — WAREHOUSE overrides REGION when both are configured with conflicting values**
**Given** `enable_switch_ticket` is REGION=`true` and WAREHOUSE=`false` for a driver's warehouse, with no DRIVER-level override,
**When** a driver in that warehouse logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to `false` (WAREHOUSE overrides REGION), and no "Edit"/Switch entry point is shown — the feature behaves as disabled.

**AC5 — DRIVER overrides WAREHOUSE when both are configured with conflicting values**
**Given** `enable_switch_ticket` is WAREHOUSE=`true` for a warehouse and DRIVER=`false` for one specific driver (Driver A) in that warehouse, while another driver (Driver B) in the same warehouse has no DRIVER-level override,
**When** Driver A logs in,
**Then** the effective flag resolves to `false` (DRIVER overrides WAREHOUSE) and no "Edit"/Switch entry point is shown;
**And when** Driver B logs in,
**Then** the effective flag resolves to `true` (falls back to WAREHOUSE) and the feature is available for Driver B, unaffected by Driver A's override.

**AC6 — DRIVER overrides REGION directly when WAREHOUSE is unset**
**Given** `enable_switch_ticket` is REGION=`false`, WAREHOUSE unset, and DRIVER=`true` for a specific driver,
**When** that driver logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to `true` (DRIVER overrides REGION, skipping the unset WAREHOUSE level), and the "Edit"/Switch feature is available despite the region-level flag being `false`.

**AC7 — DRIVER overrides both WAREHOUSE and REGION when all three conflict**
**Given** `enable_switch_ticket` is REGION=`true`, WAREHOUSE=`true`, and DRIVER=`false` for a specific driver,
**When** that driver logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to `false` (DRIVER, the most specific level, takes final precedence), and no "Edit"/Switch entry point is shown, even though REGION and WAREHOUSE are both `true`.

**AC8 — Resolves to default (`false`) when no level is configured**
**Given** `enable_switch_ticket` is not configured at REGION, WAREHOUSE, or DRIVER level for a driver,
**When** that driver logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to its default value (`false`), and no "Edit"/Switch entry point is shown.

**AC9 — Consistent resolution when all 3 levels agree (no real conflict)**
**Given** `enable_switch_ticket` is REGION=`true`, WAREHOUSE=`true`, and DRIVER=`true` (all levels agree),
**When** that driver logs in and navigates to the Booked ticket screen,
**Then** the effective flag resolves to `true` with no conflict, and the "Edit"/Switch feature is fully available.

---

## Acceptance Criteria — Switch Ticket Functional (FE)

### Entry point & button styling

**AC10 — "Book" button changes to "Edit" button on a booked ticket/route**
**Given** the driver is logged in and has successfully booked at least 1 ticket/route,
**When** the driver views the ticket/route card on the Total Booked screen and on the Booking Session (BS) screen containing the same ticket,
**Then** both cards show an "Edit" button instead of "Book", displayed with a filled green background per the final design.

**AC11 — Edit button style on Booked ticket card (Total Booked & BS screen)**
**Given** the driver has at least 1 booked ticket/route,
**When** the driver views the Edit button on the Total Booked screen and on the BS screen showing the same ticket,
**Then** the Edit button has a filled green background on both screens, consistent with the final design.

**AC12 — Detail BS (Reserved ETA): Edit + ETA buttons display correctly**
**Given** the driver has a booked ticket with a Reserved ETA, viewed via the Detail Booking Session screen,
**When** the driver navigates to that screen, taps "ETA", and taps "Edit",
**Then** the screen shows both "Edit" and "ETA" buttons, the ETA popup displays the correct reserved pickup time, and the Edit Ticket popup opens with Switch/Unbook actions.

**AC13 — Detail BS (Unreserved ETA): ETA button on group disabled, Edit enabled**
**Given** the driver has a booked ticket without a Reserved ETA, viewed via the Detail Booking Session screen,
**When** the driver navigates to that screen,
**Then** the ETA button on the group is disabled/greyed out (not tappable), while the Edit button remains enabled and opens the Edit Ticket popup as normal.

### End-to-end switch flow

**AC14 — E2E: Switch ticket from Booked ticket screen (Reserved ETA case)**
**Given** the driver has 1 ticket booked with a Reserved Pickup ETA (e.g. 10:05 AM) and at least 1 other ticket/route is available to switch to,
**When** the driver taps "Edit" on the Total Booked screen → "Switch" → "Got it" on the instructional dialog → "Book" on an available ticket/route → "Switch" to confirm (the confirmation popup summarizes the change and shows the Reserved ETA),
**Then** the switch succeeds, the driver returns to the Active Route/Other tabs with the new ticket reflected as booked, and the old ticket is no longer shown as booked.

**AC15 — E2E: Switch ticket from Detail Booking Session screen**
**Given** the driver has 1 ticket booked, accessible via the Detail Booking Session screen, and at least 1 other ticket/route is available,
**When** the driver taps "Edit" on that screen → "Switch" → "Got it" → "Book" on an available ticket/route → "Switch" to confirm,
**Then** the switch succeeds and the driver returns to the Active Route tab with the new ticket reflected as booked.

**AC16 — E2E: Switch ticket, case without Reserved ETA**
**Given** the driver has 1 ticket booked WITHOUT a Reserved Pickup ETA, and at least 1 other ticket/route is available,
**When** the driver taps "Edit" → "Switch" → "Got it" → "Book" on an available ticket/route → "Switch" to confirm,
**Then** the confirmation popup shown before confirming has no ETA time field, the switch succeeds, the driver returns to the Active Route tab with the new ticket booked, and any "Reserve Pickup ETA by [time]" text still shows if applicable to the new ticket.

**AC17 — E2E: Switch ticket with 2 tickets booked in 2 groups of the same BS**
**Given** the driver has booked 2 tickets, one in each of 2 different groups within the same Booking Session,
**When** the driver taps "Edit" on one of the 2 booked tickets → "Switch" → "Got it" (redirected to Booking Sessions screen showing other available zones, excluding the 2 already booked) → "Book" on an available ticket/route → "Switch" to confirm,
**Then** the switch succeeds for the selected ticket only, and the other booked ticket (in the other group) remains unaffected.

**AC18 — Confirmation popup shows from → to summary**
**Given** the driver is mid-way through the Switch flow and has selected a new ticket/route,
**When** the driver taps "Book" on the new ticket/route,
**Then** the Switch confirmation popup opens and clearly displays both the current (from) and new (to) ticket/route info, so the driver can verify the change before confirming.

**AC19 — Tap Cancel on Confirmation popup keeps original ticket unchanged**
**Given** the driver is on the Switch confirmation popup, having selected a new ticket/route,
**When** the driver taps "Cancel",
**Then** the popup closes, the driver returns to the Booking Session screen with no switch performed, and the original ticket remains booked and unchanged.

**AC20 — Tap-out on Dialog guide to switching dismisses without navigation**
**Given** the driver has tapped "Switch" in the Edit Ticket popup and the instructional dialog is displayed,
**When** the driver taps outside the dialog (tap-out area),
**Then** the dialog is dismissed, the driver remains on the same screen (no navigation to Booking Sessions), and the original ticket remains booked and unaffected.

### Pickup window / ETA carry-over

**AC21 — Deliver_time is carried over correctly after switch (Reserved ETA case)**
**Given** the driver has a booked ticket with a Reserved ETA and a known pickup window (e.g. 9:00–9:30 AM),
**When** the driver performs the Switch flow and views the Switch confirmation popup,
**Then** the popup displays the same Deliver_time/pickup window as the original ticket, and after confirming, the new ticket reflects the same Deliver_time/pickup window carried over.

**AC22 — Deliver_time is carried over correctly after switch (w/o Reserved ETA case)**
**Given** the driver has a booked ticket without a Reserved ETA, with a known pickup window,
**When** the driver performs the Switch flow and views the confirmation popup (w/o Reserved ETA),
**Then** the popup shows the same Deliver_time carried over from the original ticket, and after confirming, the new ticket reflects the same Deliver_time/pickup window as before switching.

**AC23 — ETA time displayed accurately on Confirmation popup (Reserved ETA)**
**Given** the driver has a booked ticket with a specific Reserved Pickup ETA (e.g. 10:05 AM),
**When** the driver starts the Switch flow and selects a new available ticket/route,
**Then** the Switch confirmation popup's ETA time field matches the original ticket's Reserved Pickup ETA exactly.

### Analytics

**AC24 — "Switch Ticket" event is logged distinctly from cancel event**
**Given** analytics/event tracking is available and the driver has a booked ticket eligible to switch,
**When** the driver performs a full successful Switch flow,
**Then** a "Switch Ticket" event is logged with the correct payload (from-ticket ID, to-ticket ID, timestamp), and no "Cancel" event is logged for this action.

**AC25 — Regression: Unbook ticket flow still works correctly and logs the correct event**
**Given** the driver has 1 ticket/route booked and analytics/event tracking is available,
**When** the driver taps "Edit" → "Unbook" → confirms,
**Then** the Edit Ticket popup shows both Switch and Unbook (unaffected by the new feature), the ticket unbooks successfully and reverts "Edit" back to "Book", the existing Unbook/cancel-type event is logged with the correct payload, and no "Switch Ticket" event is logged for this action.

### Availability & edge cases

**AC26 — Switch button disabled when no session is available**
**Given** the driver has 1 booked ticket and no other ticket/route or DB session is currently available to switch to,
**When** the driver taps "Edit",
**Then** the Edit Ticket popup opens with the "Switch" button disabled/greyed out (not tappable), while "Unbook" remains enabled and functional.

**AC27 — Switch button enabled when ≥1 session is available**
**Given** the driver has 1 booked ticket and at least 1 other ticket/route or DB session is available to switch to,
**When** the driver taps "Edit" and then "Switch",
**Then** the "Switch" button is enabled/tappable, and tapping it opens the instructional dialog as expected.

**AC28 — Edit Ticket popup displays correctly when no ticket is available to switch**
**Given** the driver has 1 ticket/route booked and no other ticket/route or DB session is currently available,
**When** the driver taps "Edit" and attempts to tap the disabled "Switch" button,
**Then** the popup opens without error/crash, "Switch" is shown in its disabled/greyed-out style per the final design, tapping it does nothing (the instructional dialog is NOT shown), "Unbook" remains enabled and fully functional, and the overall popup layout/content matches the final design for the no-availability state.

**AC29 — Ticket limit = 1, no ticket booked yet: normal booking confirmation shown**
**Given** the driver's daily ticket limit is 1 and no ticket has been booked yet,
**When** the driver taps "Book" on any available ticket/route and confirms,
**Then** the normal ticket/route booking confirmation popup is shown (not the Switch confirmation popup), the ticket books normally, and its "Book" button changes to "Edit" (per AC10).

**AC30 — Ticket limit = 0 (already booked): Book disabled on other tickets**
**Given** the driver's ticket limit is 1 and the driver has already booked 1 ticket (remaining limit = 0),
**When** the driver views other available tickets/routes and the currently booked ticket,
**Then** the "Book" button on all other tickets/routes is disabled/greyed out, while "Edit" remains available and tappable on the currently booked ticket, allowing Switch or Unbook.

**AC31 — Booking Sessions list: zone fully booked**
**Given** a zone within the Booking Session has reached full booking capacity,
**When** the driver triggers the Switch flow and navigates to the Booking Sessions screen,
**Then** the fully booked zone is shown as not selectable/disabled.

**AC32 — Booking Sessions list: zone reached max bookable count at this time**
**Given** a zone still has unbooked tickets but the driver has reached the max number of tickets bookable at the current time for that zone,
**When** the driver triggers the Switch flow and navigates to the Booking Sessions screen,
**Then** that zone/ticket option is shown as disabled, even though unbooked tickets technically remain.

**AC33 — Booking Sessions list: ≥2 zones available**
**Given** at least 2 zones within the Booking Session have available tickets/routes to switch to,
**When** the driver navigates to the Booking Sessions screen during the Switch flow,
**Then** both zones display as selectable with their available options, and tapping "Book" on an option in either zone independently proceeds to the Switch confirmation popup.

**AC34 — Selected session gets booked by another driver before confirm**
**Given** the driver has selected a new ticket/route and is on the Switch confirmation popup, and another driver books the same ticket/route before this driver confirms,
**When** the driver taps "Switch" to confirm,
**Then** a graceful error message indicates the session is no longer available, the driver is returned to the Booking Sessions screen to pick another option, and the original ticket remains booked and unaffected.

**AC35 — Zone becomes full exactly at the confirm-switch moment**
**Given** the driver has selected a new ticket/route in a zone with exactly 1 remaining slot, and another driver books that last slot right after this driver taps "Book" but before tapping "Switch" to confirm,
**When** the driver taps "Switch" to confirm,
**Then** the confirm fails gracefully with a clear error message (zone now full / session no longer available), the driver returns to the Booking Sessions screen to pick another option, and the original ticket remains booked and unaffected.

### Regression

**AC36 — Driver switches ticket multiple times (≥3) in one session**
**Given** the driver has 1 booked ticket and at least 3 other tickets/routes available to switch to sequentially,
**When** the driver performs Switch #1, immediately performs Switch #2 on the newly booked ticket, and then performs Switch #3,
**Then** each switch succeeds with no restriction or error, none of the switches count against the ticket limit, and the driver still holds exactly 1 booked ticket after all switches.

**AC37 — Unbook flow regression check (Edit Ticket popup)**
**Given** the driver has at least 1 booked ticket/route,
**When** the driver taps "Edit" → "Unbook" → confirms,
**Then** the Edit Ticket popup shows both "Switch" and "Unbook" actions, the unbook dialog is shown (existing flow, unaffected by Switch), and the ticket is unbooked successfully — same behavior as before this feature.

**AC38 — Map button regression check**
**Given** the driver has at least 1 booked ticket/route,
**When** the driver taps "Map" on the Booked ticket screen and on the Detail Booking Session screen,
**Then** the Map screen opens correctly in both cases, unaffected by the new Edit/Switch feature.

### Feature flag on/off/default

**AC39 — Switch Ticket feature is available when `enable_switch_ticket = true`**
**Given** `enable_switch_ticket` is `true` for the driver's app config and the driver has at least 1 ticket/route booked,
**When** the driver navigates to the Total Booked/BS screen, taps "Edit", taps "Switch", and completes the flow,
**Then** the "Edit" button is shown (instead of "Book"), the Edit Ticket popup opens with Switch/Unbook, and the full Switch flow works end-to-end with the switch completing successfully.

**AC40 — Switch Ticket feature is disabled/hidden when `enable_switch_ticket = false`**
**Given** `enable_switch_ticket` is `false` for the driver's app config and the driver has at least 1 ticket/route booked,
**When** the driver navigates to the Total Booked/BS screen and searches the app for any Switch-related UI,
**Then** the ticket/route card shows the original pre-feature UI with no Edit/Switch entry point shown anywhere, the existing cancel/unbook flow still works unaffected, and no Switch-related screen or action (Edit Ticket popup, instructional dialog, Switch confirmation) is reachable anywhere in the app.

**AC41 — Switch Ticket feature behavior when `enable_switch_ticket` uses its default value (unset)**
**Given** `enable_switch_ticket` is not explicitly configured for the driver's app config (falls back to its default, same as `false`) and the driver has at least 1 ticket/route booked,
**When** the driver navigates to the Total Booked/BS screen,
**Then** the app behaves identically to the `false` case — no Edit/Switch entry point is shown, the existing cancel/unbook flow still works, and the app behaves stably and predictably with no crash or inconsistent state due to the missing explicit config.

---

