# Ticket Availability Checker — Feature Logic (QA Reference)

> Source: [Confluence — Ticket Availability Checker — Feature Logic (QA Reference)](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2574385154/Ticket+Availability+Checker+Feature+Logic+QA+Reference)

## Purpose

Periodically scans active ticket booking sessions and, when a session still has **unbooked (open) tickets**, sends eligible drivers a "Tickets Available! 🎟️" notification (push or SMS) nudging them to book.

## End-to-end flow

A single, scheduled worker driven by a schedule message:

```
Scheduler (ScheduleItem message)
   └─ process()
        ├─ loadActiveBookingSession()   → pick sessions currently in their booking window
        └─ for each active session (in parallel):
             sendNotification(session, notification_type)
                 ├─ throttle / dirty-bit gate (Redis lock)
                 ├─ open-tickets gate
                 ├─ eligible-drivers gate
                 └─ publish PN (FCM) or SMS  +  tracking event
```

## Which sessions are considered

A session is included only if **all** are true (vs "now"):

1. `bookingStartTime` is **before** now − 1 hour (booking open ~1h+),
2. `targetDate` is **after** now − 24 hours (session day not more than a day past),
3. `bookingEndTime` is **null OR still in the future** (booking window not closed).

## Decision logic (per session, in order)

1. **Throttle / dirty-bit gate** — acquire the Redis lock `{environment}-BS_{bookingSessionId}:NOTIFICATION:{type}` with TTL = `throttle` (default 30 min).
   - Lock **acquired** → proceed (the throttle window starts now).
   - Lock **not acquired** (already sent within the window):
     - `check_dirty = false` (default) → **stop** (throttled).
     - `check_dirty = true` → check the dirty key `{environment}-BS_{bookingSessionId}:NOTIFICATION_DIRTY:{type}`. **That key holds a timestamp of when availability last changed.** If that change happened **more than `dirty_throttle` ago**, the worker sends now — it clears the dirty key and starts a fresh throttle window. If the change was more recent (or the key doesn't exist) → **stop**.
   - The dirty key is **set by an external producer** when availability changes; this worker only reads and clears it.
2. **Open-tickets gate** — keep only the session's tickets with no holder (**unbooked**). If none → **stop**.
3. **Eligible-drivers gate** — get matched drivers and exclude any whose status is in `NO_BOOKING_NOTIFY` (a predefined collection/set of driver statuses defined in the shared `DRIVER_STATUS` constants library). If none remain → **stop**.
4. **Send** — one message per driver, publish, then emit a tracking event.

## Throttling & idempotency

- At most one send per `session + notification_type` per `throttle` window (default **30 min**), enforced by the Redis lock TTL — no DB flag.
- A failed acquire does **not** extend the window; it expires on its original schedule.
- The lock is taken **before** the open-ticket/driver checks, so a run that wins the lock but finds nothing to send still consumes the throttle window.
- The **dirty-bit** path (`check_dirty = true`) only does something if an external producer is actually stamping the dirty key — otherwise it behaves exactly like plain throttling.
- Sessions are processed concurrently.

## Notification content

Depends on `notification_type` (default `PN`):

**PN (push / FCM):**

- Title `Tickets Available! 🎟️`, message "Book your tickets now"
- `category = ASSIGNMENT_DISTRIBUTOR`, `action = NOTIFY_AVAILABLE`
- `isPromotional = true` (subject to per-driver promo rate limits), `saveAsAnnouncement = true`
- body = rendered `message_template`
- data payload: `schedule_id`, `action = assignment_available`, `screen = Schedule`

**SMS:**

- Body = rendered `message_template`, sent to the driver's phone, `isPromotional = true`.

**Template variables** for `message_template`: `${first_name}`, `${last_name}`, `${available_group}` (comma-separated open ticket names), `${available_ticket}` (open ticket count), `${session_name}`.

## FCM delivery & storage

The worker only publishes the message; a separate FCM consumer delivers it and handles storage. Two records result:

- **In-app announcement** — because `saveAsAnnouncement = true`, the notification is saved to the driver's announcement / notification-history store, so the driver can view it later in the app (not just as a one-time push). This is where the content lives.
- **Tracking event** — the worker also emits an event recording the *send* (notification type, driver count, message template, booking session). It does **not** store the full push payload; it's for analytics/audit.

The extra push fields are **not for storage — they drive handling and navigation**:

- **`category`** — classifies the notification for grouping/channel on the device and in the announcement list.
- **`action`** — an identifier the consumer and the driver app use to decide how to handle the notification.
- **`data` payload** — values the app reads to deep-link when tapped: `screen = Schedule` (open the Schedule screen), `schedule_id` (which booking session), `action = assignment_available` (in-app behavior).

In short: `title`/`message` are what the driver sees; `category`/`action`/`data` are what the system uses to route, persist, and deep-link it.

## Why a driver may not receive the push

Even after the worker sends the notification, the delivery layer may suppress it for a specific driver. These are **expected, not errors**. You can find the reason on Datadog (`service:notification-service`) by searching the message content:

- **Driver unsubscribed from marketing push** — `Don't send PN because driver {id} unsubscribe Marketing PN with message {message}`
- **Driver reached the promotional notification limit** — `Don't send PN because driver {id} reach limit with message {message}`

## Configurable parameters

`notification_type`, `throttle`, `check_dirty`, `dirty_throttle`, and `message_template` are **read from the schedule message's attributes (`schedule.attributes`)** — i.e. set in the scheduler/cron definition that fires this worker. `enable_batch_fcm` is a worker config attribute from Consul.

| Parameter | Source | Meaning | Default |
| --- | --- | --- | --- |
| `notification_type` | schedule.attributes | `PN` (push) or `SMS` | `PN` |
| `throttle` | schedule.attributes | Lock TTL = min gap between sends per session+type (ms) | 1,800,000 (30 min) |
| `check_dirty` | schedule.attributes | Enable the dirty-bit early-resend fallback | false |
| `dirty_throttle` | schedule.attributes | How far in the past the availability change must be to trigger a re-send (ms) | 300,000 (5 min) |
| `message_template` | schedule.attributes | Notification body, with variable substitution | "Ticket are still available." |
| `enable_batch_fcm` | Consul KV `apps.workers.fcm.enable_batch_fcm` (read at startup) | Batch all PN messages into one publish | false |

## Behavior notes / known limitations

> [!NOTE]
> - **Throttle is the dedupe mechanism** — no per-session "already sent" DB flag; too-low `throttle` ⇒ frequent re-notifications.
> - **Recurring nudge by design** — as long as unbooked tickets remain and the throttle has elapsed, the session is notified again on the next run.
> - **Lock consumed without sending** — a run that wins the lock but finds no open tickets / no eligible drivers still holds the throttle window for the full `throttle` duration.
> - **No open tickets / no eligible drivers ⇒ silent stop**, no event emitted.
> - **`enable_batch_fcm` is read once at startup and is a shared FCM key** — changing it requires a restart and affects all FCM-sending workers; the value must be exactly `true`.
> - **Dirty bit depends on an external producer** — if nothing stamps the dirty key, `check_dirty = true` behaves like plain throttling.