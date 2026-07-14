# [Pickup ETA] Related events

> Source: [Confluence — \[Pickup ETA\] Related events](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1756397574/Pickup+ETA+Related+events)

This document lists some events related to the Pickup ETA flow.

## 1. Assignment

**Scenario:**

- Driver books ticket + reserves ETA, then claims route successfully.
- Dispatcher does an action → Expectation: on the `ticket` collection, the value of `item` and `status` is reset to null OR removed.

| Events | How to create the event |
| --- | --- |
| `ASSIGNMENT.OUTBOUND.reassign` | On Assignment details (driver already claimed) → click Reassign to another driver |
| `ASSIGNMENT.OUTBOUND.unassign` | On Assignment details (driver already claimed) → Unassign driver |
| `ASSIGNMENT.OUTBOUND.rejected` | On Driver App: driver declines route then waits for assistant. On Outbound App: staff retrieves the declined route |

## 2. Ticket

**Scenario:**

- Booking session has config `booking_session.attributes.enable_pickup_secondary: true` (the 2nd booked tickets within the BS are not required to have an ETA).
- Driver has 2 booked tickets AND the 2nd ticket has no ETA.
- → when the 1st ticket is void/discard/unbook/reassign → `TICKET.PLANNING.trigger_require_eta` is logged.

| Events | How to create the event |
| --- | --- |
| `TICKET.PLANNING.void` | On Dispatch App → Booking session page; Dispatcher voids the 1st booked ticket |
| `TICKET.PLANNING.discard` | On Dispatch App → Booking session page; Dispatcher deletes/discards the 1st booked ticket |
| `TICKET.PLANNING.unbook` | On Driver App → Driver unbooks the 1st booked ticket |
| `TICKET.PLANNING.reassign` | On Dispatch App → Booking session page; Dispatcher reassigns the 1st booked ticket to another driver |
