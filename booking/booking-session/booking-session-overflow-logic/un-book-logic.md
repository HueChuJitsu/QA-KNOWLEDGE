# Un-book logic

> Source: [Confluence — Un-book logic](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/852689012/Un-book+logic)

## 1. Rebook restriction after unbook (within a ticket group)

After a specific period of time, a driver cannot rebook a ticket in the same ticket group if they have previously unbooked / voided / automatically unbooked a ticket belonging to that ticket group. This info is saved in the collection `booking_session`:

```json
"unbook_penalty": {
  "enable": true,
  "duration_to_unbook": "12h",
  "penalty_to_book_in_seconds": 300.0
}
```

or

```
"attributes.unbook_penalty_limit": "14h"
```

- The `unbook_penalty` field has higher priority than the `unbook_penalty_limit` field.
- _Updated from 07/12/2023:_ No longer use values in `booking_session.attributes`; only check the penalty set up in `booking_session.unbook_penalty`.
- `duration_to_unbook` meaning: if the driver books and unbooks a ticket within `{duration_to_unbook hour + booking start time}`, the driver is not penalized. If out of that time range, the driver is penalized by not being able to be booked for `penalty_to_book_in_seconds`. Penalty info is saved on Redis.
- _Updated first half of 2026:_ Changed to penalize the driver by not allowing them to rebook any ticket/route on the same day as the unbooked ticket/route — not only in the same ticket zone as before.
  - New Redis key: `{env}-LOCK:UNBOOK:DR_{driverID}_TargetDate:{BookingTargetDateTime UTC back to beginning of Date}`

## 2. Unbook after N minutes → un-book penalty driver crew

If the driver unbooks after N minutes from the book, the driver is added to the "un-book penalty driver crew" and can't book the booking session of the next day. They are removed from the crew when completing a route.

- N minutes is configured in [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1/kv/staging/apps/workers/event_processors/assignment/attributes/unbook_time_no_penalty/edit).
- The list of driver crews per region is configured in [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1/kv/staging/apps/workers/event_processors/assignment/attributes/unbook_crew_ids/edit).

## 3. Claim restriction

Drivers cannot unbook after claiming a ticket.
