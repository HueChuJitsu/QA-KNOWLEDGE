# Remind Ticket Popup — Feature Logic (QA Reference)

> Source: [Confluence — Remind Ticket Popup — Feature Logic (QA Reference)](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2571829253/Remind+Ticket+Popup+Feature+Logic+QA+Reference)

## Purpose

Sends a push notification ("Upcoming Routes Reminder" popup) to drivers who already hold a ticket for an upcoming booking session, reminding them to show up on time. The popup carries action `TICKET_REMIND` and includes confirm / unbook buttons.

## End-to-end flow

The reminder is produced by a **two-worker chain** — it is never sent on its own timer:

```
AdvanceTicketBookingRefreshWorker   (scheduled)
   └─ reads `notification_only` from the schedule message
   └─ matches the booking session for the target day
   └─ publishes a BookingAnnouncementMessage (type = ADVANCE) → ANNOUNCEMENT exchange
        ONLY when ALL conditions are met:
          • session status = IN_PROGRESS
          AND
          • session HAS a booking model  AND notification_only = true
            — or —
            session has NO booking model AND notification_only = false (not in use)
                                  │
                                  ▼
BookingSessionAnnouncementWorker   (consumes the message)
   └─ remindDriverHaveTicket()  → THE REMIND TICKET POPUP
```

## Decision logic (within the remind step)

When the announcement worker processes a message, the reminder is sent **only if every condition below passes**, checked in this order. If any fails, the reminder is skipped (logged) and processing stops.

1. **Session exists** — the booking session referenced by the message is found.
2. **Region enabled** — `remind_ticket_enable_regions` contains `ALL`, or the session's region is in the list.
3. **Not already sent** — the session's `sent_remind_ticket` flag is not yet `true` (prevents duplicates; one reminder per session).
4. **Timing reached** — the current time is at or after the computed send time `remindAt` (see formula below).
5. **Has recipients** — at least one ticket in the session has an assigned driver (holder). If none, it stops *without* marking sent, so it can retry on a later message.

If all pass, it builds one popup per driver, publishes them (individually or batched), then sets `sent_remind_ticket = true` so it won't send again.

## Send-time calculation (`remindAt`)

```
remindAt = session targetDate
            → converted to the session's timezone
            → minus  remind_ticket_days_before  (default 1 day)
            → start of that day (00:00 local)
            → plus   remind_ticket_send_time     (default PT18H → 18:00 local)
```

- All math is done in the **session's own timezone**, not the server's.
- **Example:** target date `2026-06-13` (in session tz), `days_before = 1`, `send_time = 18:00` → reminder eligible from **2026-06-12 18:00 local** onward.
- The reminder fires the first time an announcement message is processed at/after `remindAt`.

## Message content

- **Title:** `remind_ticket_title` (default "Upcoming Routes Reminder").
- **Body:** uses the **with-ETA** template if the driver's ticket has an ETA (`validFrom`) — personalized with first name, ETA time, and session name; otherwise the **no-ETA** template.
- If a driver holds multiple tickets, the ETA shown is the **earliest** one.
- **Buttons:** positive ("I'll be there") and negative ("I won't be able to make it").
- **Unbook confirmation:** tapping the negative button shows a reconfirm dialog (`reconfirm_title`, `reconfirm_message`, and reconfirm buttons).
- The notification is also saved to the driver's in-app notification history.

## Configurable parameters

| Parameter | Meaning | Default |
| --- | --- | --- |
| `remind_ticket_enable_regions` | Regions where reminders are active (`ALL` = everywhere) | none (off) |
| `remind_ticket_days_before` | How many days before the session day to send | 1 |
| `remind_ticket_send_time` | Time-of-day offset (ISO-8601 Period, e.g. `PT18H` = 18:00) | PT18H |
| `remind_ticket_title` | Popup title | "Upcoming Routes Reminder" |
| `remind_ticket_message_with_eta` / `_no_eta` | Body templates | see app |
| `positive_button` / `negative_button` | Button labels | "I'll be there" / "I won't be able to make it" |
| `reconfirm_title` / `reconfirm_message` / `reconfirm_positive_button` / `reconfirm_negative_button` | Unbook confirmation dialog | see app |

**Config behavior:**

- These are read from **Consul** at runtime — changing them does **not** require a service restart; updates are picked up within ~5 minutes.
- `remind_ticket_enable_regions` is stored in a different config section than the rest, so if regions are configured but reminders never send, that key's placement is the first thing to check.

## Behavior notes / known limitations (for QA awareness)

> [!NOTE]
> - **One reminder per session.** Once sent, `sent_remind_ticket` blocks any repeat, even across multiple incoming messages.
> - **No recipients ≠ failure.** If no driver holds a ticket yet, it quietly skips and remains eligible to send on a later message.
> - **Send-failure caveat:** if the push fails to publish, the session is still marked as sent — the reminder is not automatically retried.
> - **No session-status check** in the remind step itself: if a reminder message is reprocessed for a session that is no longer active, the popup can still go out. Treat an unexpected reminder for a cancelled/closed session as a defect.
> - **Date semantics:** the "day before" depends on how `targetDate` is stored relative to the session timezone; verify reminders land on the intended calendar day.