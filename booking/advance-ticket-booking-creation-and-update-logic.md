# Advance ticket booking creation and update logic

> Source: [Confluence — Advance ticket booking creation and update logic](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/845119492/Advance+ticket+booking+creation+and+update+logic)

_This document is to note the logic of advance ticket booking._

## 1. Difference between Advance booking and regular booking (**outdated**)

| **Advance booking session** | **Regular booking session** |
| --- | --- |
| Having Booking Forecast | Not having Booking Forecast |
| Created before Target date 2 days | Created after selecting solutions |
| Can create without Assignments | Only can be created when already having Assignments |
| Driver will not be added when the booking session is created. Driver is added 2 times: once at 15:00 local time on the day the BS is created, once at 15:00 local time the next day (driver crew from `refresh_advance_booking_config`) | Driver is added when the booking session is created (from `booking_session_template.crews`) |
| Number of tickets can differ from number of assignments in the BS | Number of tickets = number of assignments in the BS |

## 2. Create Advance booking

### 2.1. On production

The Advance booking Schedule calls a Worker to create the Advance booking session.

- Booking session is created based on the Refresh config set up in `schedule.attributes`. Refresh config data is saved in collection `refresh_advance_booking_config`.
- **Type = booking:**
  - `booking_start_time`: booking session is created with that booking start time. Booking session start time = Booking session target date (timezone of refresh config) snapped to beginning of day + `booking_start_time` in refresh config, on the respective day.
  - `day_in_advance = 2` → if Firing time = today → BS created will have target date = today + 2.
  - `forecast_in_advance` → if Firing time = today → BS created will have booking forecast with date = today + 2. (**outdated**)
  - `template_id`: Id of booking template used to create the booking session.
- **Type = drivers:** (**outdated**)
  - `crew_ids`: list of driver crews added to the booking session upon creating.
- Created Booking session will have a `booking_end_time`.
- Created booking session will include booking zones which satisfy one of the two conditions:
  - Booking zone is set up with param `empty_allowed = true` in the booking template.
  - Booking zone is set up with number of tickets (`average_ticket`) > 0.
- Created booking session contains the driver crew set of the Refresh config (**Outdated — driver will be from `booking_model` logic**).
  - The list of driver crews in the booking session is refreshed when schedule `Ticket_refresh` runs, following the refresh config `schedule.attributes`.

On booking session target date −2, when the schedule runs (or its Worker runs), the 5 driver crews in the step with `day_in_advance = 2` are added to the booking session and `booking_start_time` of the booking session is reset to 18:00 of the Target date. The next day, the 5 driver crews in the step with `day_in_advance = 1` are added. The driver crews are the same, as we want to update the latest drivers in each driver crew to the booking session.

### 2.2. Another way to create a booking session (**Outdated**)

Forecast screen → select forecast of target date → click **[Create booking session]** → select booking template → click **[Create]**.

**Notes:**

- Each Booking template can only be used once per date → cannot have 2 booking sessions with the same target date and the same booking template.
- Conditions for the list of booking templates in the Booking forecast screen: same region and warehouse as the booking forecast.

## 3. Add Booking zone manually

Open Booking session → click **[Add new ticket book group]** icon → the "Add ticket book group" popup is displayed.

- The list of zones to be added is retrieved from ticket books not in the booking session in the booking template, from the first booking session template of the BS's region found (no special order, just from the DB finding result).
- Other information in the "Add ticket book group" popup is retrieved from `forecast.ticket_zone`. Those fields (except Warehouse) are editable, so we must set up the warehouse id in `forecast.ticket_zone` if we want to manually add that zone to the booking session.

## 4. Adding assignments to a booking session

### 4.1. Automatically

By `RefreshBookingSessionWorker` (runs after selecting a solution; time set up in [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1/kv/staging/apps/workers/booking_session_refresh/attributes/delay_in_seconds/edit); related [Datadog](https://app.datadoghq.com/logs?query=service%3Aworker-booking-session-refresher)) or by the **[Refresh Assignment]** icon.

Conditions to add an assignment to the respective zone, based on the Booking template:

- Client of assignment is Commingle.
- Assignments satisfy the conditions in the Booking template.

### 4.2. Manually add Assignment to ticket book

Adding an assignment here just updates `ticket_book.min/max ocac`, but the UI currently reads from `min/max ccac` to display.

## 5. Related tables / collections

`booking_session`, `booking_session_template`, `booking_forecast_v2`, `refresh_advance_booking_config`, `ticket_book`, `ticket` (a type-assignment is a ticket in the Booking session).

## 6. Config

To be able to create a BS with the v2 flow, turn on the corresponding `item_metadata` config.
