# Reserve pickup ETA

> Source: [Confluence — Reserve pickup ETA](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/845906005/Reserve+pickup+ETA)

This document is to provide information about the Reserve pickup ETA feature.

Related documents:

- [ENG-738689025](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/738689025)
- [ENG-647823383](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/647823383)
- [ENG-647692354](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/647692354)
- New Pickup session flow → [Dynamic Pickup ETA: Configuration](new-pickup-session-flow/dynamic-pickup-eta-configuration.md)

## Notes

- Pickup eta (pickup slot) is the time slot that a driver intends to come to pickup their shipments → this helps Outbound staff to prepare shipments better when drivers come.
- **Ticket booking tutorial logic:** default 6 pickup eta slots, each slot lasts 30 mins. Start time of the list of ETA slots is the time the shipment is scanned to create the ticket booking tutorial. Date of the ETA slot is the date the tutorial is created. Timezone is the timezone of the driver region (from the latest assignment).
- **Where to set up pickup slot for Real Ticket Booking:** Dashboard → Warehouse (**Outdated** — refer to [Dynamic Pickup ETA: Configuration](new-pickup-session-flow/dynamic-pickup-eta-configuration.md)).
  - **Quantity per Pickup Slot:** for each pickup slot, how many drivers can come. A pickup slot here is for each ticket book → for 2 different ticket books, max 4 drivers can select pickup slot 3:00–3:30 (based on example data).
  - **Pickup Time Gap:** if pickup time gap = 2 → if a Driver selects a pickup slot, that slot is disabled along with the 2 pickup slots above and below.
  - **Time per Pickup Slot:** how long a pickup slot lasts.
  - **Earliest / Latest Pickup Hour:** time range of pickup slots. Can set a default value to apply for all days, or set for each separate date to overwrite the default value.

- **(Outdated)** When a driver selects a pickup slot, he is considered to link to a ticket pickup slot. The ticket pickup slot also links to the driver's ticket (main ticket, which the driver books to be able to claim assignment).
  - Database: collection `ticket`, type `pickup_slot`.
  - **Q: How to find a ticket pickup slot in the database?**
    A: Get `booking_session.groups.attributes.pickup_session` → query in collection `booking_session` with `id = pickup_session` → get `booking_session.groups.items` → search in collection `ticket` with the ids found in the items section.
  - **Q: Which logic currently uses the pickup slot ticket?**
    A: To check if all pickup slots are past, to allow a driver to book and proceed with a ticket without selecting an eta. If current time > `from_ts` of all pickup slot tickets (in timezone of booking session), all pickup slots are past.
- If `booking_session.session_attributes.can_filter_outdated_pickup_slot = true`, pickup slots already past will not be displayed on the Driver app anymore.
- If a driver is set up to require eta but does not book an eta, there is a notification to remind the driver to book, and the ticket is voided if the driver doesn't book the eta within the limit time. All times are set up on [Consul](https://consul.staging.gojitsu.com/ui/dc1/kv/staging/ticket/reserve_pickup_eta/).
- If a Driver is required to pickup eta but all pickup slots are past or all pickup slots are full → the Driver is no longer required to pickup eta. The Driver views the booked ticket in the main screen in the Driver app with text "I'm on my way", not "Reserve Pickup Eta".
- **When `booking_session.session_attributes.enable_pickup_secondary` is enabled:**
  - If a driver is required to book eta: the driver books ticket 1, the system requires the driver to book eta for ticket 1 → the system will not require eta for the next ticket the driver books until the driver completes ticket 1. If the secondary function is not enabled, the driver is required to book eta for every ticket they book.
  - The driver sees the list of etas from Earliest Pickup Hour to Latest Pickup Hour before they complete booking the first ticket they own. After completing the first one, the list of etas is expanded, from Earliest Pickup Hour to Secondary Pickup Hour.

## `pickup_after_booking` flag in attribute (**Outdated**)

- If `pickup_after_booking` is set as `PT<={number of minutes}M`: it means the driver can book ETA after booking the ticket.
- For example:
  - `pickup_after_booking: PT-30` — driver can book ETA immediately after booking the ticket.
  - `pickup_after_booking: PT30` — driver can book ETA 30 mins after booking the ticket.
