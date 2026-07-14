# Dynamic Pickup ETA: Configuration

> Source: [Confluence — Dynamic Pickup ETA: Configuration](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1766522911/Dynamic+Pickup+ETA+Configuration)

This new Pickup session feature allows users to easily update the Pickup window or Pickup slot of a Booking session / Direct booking, with new collections in use, even after the Pickup session is created.

## 1. How to enable the new Pickup session feature?

Only at the warehouse level.

## 2. Related collections

- `pickup_template`
- `pickup_session`
- `pickup_window`
- `pickup_slot`

## 3. Pickup ETA flow logic

### 3.1. Pickup template

Users can create a Pickup template for each warehouse in each region in the [Dashboard](https://dashboard.staging.gojitsu.com/pickup-eta). A created pickup template has type **Main** and is created based on the Pickup template type **Default**'s `day` and `window_type_definitions`.

### 3.2. Pickup session creation logic

- When creating a Booking session (by Worker or manually), the system checks if the target date of the booking session has a Pickup session type **Main** for the respective warehouse of the Booking session zones. If it does, the system creates a Pickup session type **Sub** from that Main one. If not, the system creates a Pickup session type Main and then type Sub for that warehouse.
- When creating a Direct booking (by worker) or adding assignment/solutions (**Implementing**), the system checks if the target date of the booking session has a Pickup session type Main for the respective warehouse of Assignments. If it does, the system stops the action. If not, the system creates a Pickup session type Main for that warehouse and date.
- A Pickup session type Main is expected to manage all pickup windows and slots for all Booking sessions and Direct Bookings in the current date.
- A Sub Pickup session is saved in `booking_session.groups.attributes.pickup_session` as before, and has a corresponding record in `item_metadata`.

### 3.3. Booking ETA logic

- When booking a ticket, the system checks the pickup session of the zone of the ticket → if there is a record in `item_metadata` with `owner = pickup_session`, the driver follows the new pickup ETA flow. Otherwise, the driver follows the old ETA flow.
- When booking a route, the system always follows the new pickup ETA flow. The system gets ETA data based on the Direct booking date and the assignment's warehouse.
- A driver in Direct booking can book an ETA with `composition = Direct`, while a driver in a Booking session can book an ETA with `composition = IC` or `Large` based on `booking_session.ticket_type` (if not present, checking `booking_session.attributes.ticket_type`).
- A driver also has a gap regardless of the window type they book.
  - ex: Driver books ETA slot 4:30–5:00 for a Direct booking or any Booking session → with gap 3, in both booking session and direct booking they can only book the next window 6:30–7:00.
- A Dispatcher can enable/disable a Pickup window or update the available Pickup slots of a window. If a driver already booked (not yet claimed or completed the ticket) and the dispatcher disables the window or updates available pickup slots < booked slots → the system notifies the driver of losing their ETA and allows them to rebook within a time set up in [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/ticket/reserve_pickup_eta/seconds_buffer_for_day_of_pickup_window_cancellation/edit). If they do not rebook, their ticket is voided automatically by the Reserve ETA worker; and if they are in Pool_1 or Pool_2, they are added to the Highest_priority pool.
- Other logic like pickup secondary or require ETA remains as current.

## 4. Configure user permission

Configure user permission on Prod at [acl.gojitsu.com/rules/summary](https://acl.gojitsu.com/rules/summary) with key `not_allow_update_pickup_session`.

Add a user group by clicking **Edit permitted groups**.
