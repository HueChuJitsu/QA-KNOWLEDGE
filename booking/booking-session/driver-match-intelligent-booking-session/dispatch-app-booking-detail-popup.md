# Dispatch App - Booking Detail popup

> Source: [Confluence — Dispatch App - Booking Detail popup](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1011712009/Dispatch+App+-+Booking+Detail+popup)

This document provides information about the **[Booking detail]** popup for a group of an Intelligent Booking Session.

→ When a BS is an Intelligent BS → there is an added **[Booking details]** option for each group/zone → click it to view the Booking detail popup:

- **API `getBookingZoneInfo`** returns information of the group and the list of matched drivers correspondingly.

→ The "get booking group detail" API filters out all drivers who are active to show the BS on the Dispatch App.

- **List of drivers:** drivers who have a preferZone equal to the current viewing group and belong to `booking_model.active_pools` (and drivers enabled for the new settings of Intelligent Booking at `item_metadata`).
- **Booking limit:** compares `booking_model.max_reservation` of the wave with `booking_session.booking_limit`; the smaller value takes precedence and is shown here.
- **Number of tickets:** the number of total open tickets for the wave (based on total tickets of the group × `wave.num_ticket_ratio`).
