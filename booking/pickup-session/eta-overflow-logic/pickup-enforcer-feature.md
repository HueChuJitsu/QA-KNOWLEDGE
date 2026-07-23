# Pickup Enforcer Feature

> Source: [Confluence — Pickup Enforcer Feature](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/845774956/Pickup+Enforcer+Feature)

## Purpose

For today's ticket booking sessions, it enforces that drivers who hold a ticket actually **choose a pickup time window** and **honor it**. Drivers who never picked a window ("no-pickup") or who missed their chosen window ("late-pickup") are first **warned by SMS**, and — after a grace period — have their **tickets voided**.

**Not to be confused with** `OnTimePickupEnforcer` — that's a separate worker that unassigns _routes/assignments_ when a driver doesn't physically arrive by the booked ETA. This doc covers only the **ticket pickup-time** enforcer (`pickup_enforcer.yaml`).

## ⏰ Timezone reference (read first)

| Time value | Timezone basis |
| --- | --- |
| Session selection — "target date is today" | **Server / JVM default timezone (GMT/UTC)** |
| No-pickup daily cutoff (`nopickup_limit`) | **Booking-session timezone** (start-of-day in `session.timezone`; fallback `targetDate − 12h` if no session timezone) |
| `void_after_booking` grace | **Absolute/real time** (epoch instant — timezone-independent) |
| Late-pickup window expiry (`validTo`) | **Absolute/real time** (epoch instant) |
| `no_pickup_gracious_time` / `late_gracious_time` | **Absolute/real time** (elapsed since `warned_at`) |

⚠️ Note the mismatch: which sessions are _picked up_ for processing is decided in **server/GMT** time, but the no-pickup cutoff within a session is evaluated in the **session's own timezone**. For a session whose timezone differs from the server, "today" can mean two slightly different day boundaries.

## End-to-end flow

A single, scheduled worker driven by a schedule message:

```
Scheduler (ScheduleItem message)
   └─ process()
        ├─ loadActiveSession()  → active sessions whose target date is TODAY (server/GMT tz)
        ├─ (optional) filter by `sources`
        └─ for each session (in parallel): processBookingSession()
             ├─ build candidate tickets
             ├─ No-pickup branch  → warn / void
             ├─ Late-pickup branch → ignore / warn / void
             └─ send SMS + void tickets + emit tracking events
```

## Which sessions & tickets are considered

**Sessions:** returned by `listActive()` — i.e. **active (in-progress) booking sessions** — further filtered to those whose `targetDate` falls in **today, measured in the server/JVM default timezone (GMT/UTC)**. Optional `sources` payload filters by the session's `source_uri` prefix.

**Candidate tickets** (a ticket must meet all):

- pickup enforcement is **enabled** — ticket attribute `enforce_pickup_time`, falling back to the session's `enforce_pickup_time`, else off;
- has an assigned **holder** (driver);
- status is empty **or not** `CLAIMED` **/** `COMPLETED` (i.e. not already claimed or completed);
- past the **post-booking grace** — booked more than `void_after_booking` (default 30 min) ago _(absolute/real time)_.

## Decision logic — two branches

**Warn-then-void ordering:** for either branch, if the warning flag is **on**, a ticket **must be warned first** and can only be voided **after** its gracious time elapses following that warning. Voiding without a prior warning happens **only** when the warning flag is off.

### A. No-pickup-time (driver never selected a window — `validTo` is empty)

1. **Time gate:** only evaluated once the daily cutoff has passed — `start-of-day + nopickup_limit` (default **9h** → 9:00 AM), computed in the **booking-session timezone** (fallback `targetDate − 12h` if the session has no timezone). Before that, nothing happens.
2. **Candidates:** ticket has no chosen window, and the driver doesn't already have another ticket that's `CLAIMED`/`COMPLETED` or already has a time.
3. **Warn** (only if `warn_no_pickup = true`): ticket not already warned, **and** the driver hasn't booked any ETA, **and** it isn't already too late per the Time gate condition.
4. **Void** (only if `void_no_pickup = true`): either warnings are off, **or** the ticket was warned and `warned_at + no_pickup_gracious_time` (default **1h**) has passed _(absolute/real time since the warning)_.

### B. Late-pickup (driver chose a window but it expired — `validTo` is in the past _(absolute/real time)_)

1. **Candidates:** window expired, status not `READY`.
2. **Ignore path:** if the driver is **in progress** — defined as holding **any ticket with status** `CLAIMED` — the late ticket is **not voided**; an "ignore" event is emitted instead.
3. **Warn** (only if `warn_late_pickup = true`): ticket not already warned.
4. **Void** (only if `void_late_pickup = true`): either warnings are off, **or** the ticket was warned and `warned_at + late_gracious_time` (default **30m**) has passed _(absolute/real time since the warning)_.

## Actions / outputs (only when `test = false`)

- **SMS warnings** — one per driver (no-pickup and late-pickup use separate templates; a driver warned in the no-pickup group is not also warned in the late group).
- **Void tickets** — no-pickup voids re-check first and **abort** if the driver has since booked an ETA or is too late; late-pickup voids don't re-check. Void goes through the new booking gRPC API when applicable, else the legacy path.
- **Ignore events** — for late tickets held by in-progress drivers (no void).
- **Tracking events** — emitted for voids, warnings (as `sms`), and ignores.
- **Session re-initialize** — if anything was voided, the booking session is re-initialized so freed tickets become available again.

## Idempotency & safety

- State is kept on the **ticket** (no Redis lock):
  - `warned` (= holder) + `warned_at` (**epoch millis, absolute time**) → prevents re-warning and drives the grace-then-void timing. A ticket only counts as "warned" if `warned` matches the **current** holder.
  - `ignored_late_pickup` (= holder) → prevents duplicate ignore events.
- **Dry-run:** `test = true` computes and logs every decision but sends nothing and voids nothing — the safe way to preview impact.
- Sessions are processed concurrently.

## Configurable parameters

Behavioral params come from the **schedule message attributes** (`schedule.attributes`); message copy comes from **Consul dynamicAttributes**.

| Parameter | Source | Meaning | Default | Timezone basis |
| --- | --- | --- | --- | --- |
| `sources` | schedule.attributes | Restrict to sessions whose `source_uri` prefix is in this list | (all) | — |
| `nopickup_limit` | schedule.attributes | Daily cutoff (from start of day) before no-pickup tickets are actionable | 9h | **Booking-session tz** |
| `warn_no_pickup` / `warn_late_pickup` | schedule.attributes | Enable warnings for each branch | false | — |
| `void_no_pickup` / `void_late_pickup` | schedule.attributes | Enable voiding for each branch | false | — |
| `void_after_booking` | schedule.attributes | Grace right after booking before a ticket is eligible | 30m | Absolute/real time |
| `no_pickup_gracious_time` | schedule.attributes | Delay between warning and voiding (no-pickup) | 1h | Absolute/real time |
| `late_gracious_time` | schedule.attributes | Delay between warning and voiding (late-pickup) | 30m | Absolute/real time |
| `test` | schedule.attributes | Dry run — log only, no side effects | false | — |
| `message_warn_no_pickup_time` | Consul dynamicAttributes | Warning SMS (no-pickup) | English default | — |
| `message_warn_late_pickup_time` | Consul dynamicAttributes | Warning SMS (late-pickup) | English default | — |
| `message_notify_cancel_no_pickup_time` | Consul dynamicAttributes | Void SMS (no-pickup) | English default | — |
| `message_notify_cancel_late_pickup_time` | Consul dynamicAttributes | Void SMS (late-pickup) | English default | — |

## Behavior notes / known limitations

- **Enforcement is opt-in** — nothing happens unless `enforce_pickup_time` is set on the ticket or session, and the relevant `warn_*` / `void_*` flags are enabled. All flags default to off, so a misconfigured schedule silently does nothing.
- **Warn and void are independent toggles** — voiding with warnings off skips the grace period entirely.
- **Timezone mismatch** — session selection uses server/GMT "today" while the no-pickup cutoff uses the session timezone; for non-GMT sessions this can shift day boundaries. Grace/expiry checks are absolute time and unaffected.
- **In-progress drivers are protected** from late-pickup voids (they get an ignore event instead).
- **No-pickup voids can self-abort** if the driver booked an ETA or became too late between evaluation and voiding.
- **Only today's sessions** are considered (server/GMT "today" for selection).
- **The run schedule and the live flag values are NOT stored in this repo.** The cron cadence and the actual per-run values of `nopickup_limit`, `warn_*`, `void_*`, gracious times, `test`, etc. live in the **external scheduler service / Consul**. To know how the job is actually configured in a given environment (dev/staging/prod), QA must inspect the scheduler/Consul there — the repo only shows the parameter names and their code defaults.
