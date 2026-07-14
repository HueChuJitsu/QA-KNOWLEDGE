# Max booking session number in a region

> Source: [Confluence — Max booking session number in a region](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1606811690/Max+booking+session+number+in+a+region)

`max_booking_sessions`:

- max booking sessions for a region
- configured in `item_metadata` with `owner = RG_{region}` and `scope = BOOKING`

Behavior by component:

- **Advance Booking Processor / API**
  - Can create a number of booking sessions up to `max_booking_sessions`, as long as they don't use the same `booking_session_template`.
- **Booking Session Refresher** (runs after selecting a solution)
  - Can refresh a number of booking sessions up to `max_booking_sessions`.
  - If the worker finds the number of created booking sessions is greater than `max_booking_sessions`, the worker aborts.
- **Advance ticket booking refresh** (runs at a scheduled time, triggered by Schedule `Ticket_Refresh`)
  - If `refresh_advance_booking_config` has a `template_id`, this worker refreshes only booking sessions with the same `template_id` in the config.
  - If there is more than one booking session with the same `template_id`, the worker aborts.
