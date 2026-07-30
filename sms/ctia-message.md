# CTIA Message

> Source: [CTIA message — Confluence](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/865501321/CTIA+message)

## 1. Description

A **CTIA message** is the message auto-replied to a sender when they text **HELP**, **START**, or **STOP**.

The template is configured in `telephonies.sms_macro`.

## 2. Template variables

`sms_macro` has 3 variables:

| Variable | Source |
|---|---|
| `${company_name}` | Client / client profile company name |
| `${support_email}` | Client / client profile support email |
| `${support_phone_number}` | Client / client profile support phone |

### How each variable is resolved

The data shown depends on how many **active shipments** the sender has.

**1 active shipment** → show that client's (or client profile's) company name, email, and phone:

| Variable | Logic |
|---|---|
| `${company_name}` | If `client_profiles.is_use_profile_name = true` → `client_profiles.name`, else `clients.name` |
| `${support_email}` | If `client_profiles.is_use_profile_email = true` → `client_profiles.email`, else `clients.support_email` |
| `${support_phone_number}` | If `client_profiles.is_use_profile_phone = true` → `client_profiles.phone_number`, else `clients.support_phone_number` |

If no data is set on `client_profiles` / `clients`, the system falls back to the **default** support email/phone from Consul config:
`https://consul-staging.axlehire.sdm.network/ui/dc1/kv/staging/apps/workers/incoming_sms`

**Multiple active shipments** → show a **neutral** message (no company name).

> Example (START reply): *"Thanks for subscribing to notifications! You can reply STOP to unsubscribe or HELP for more information (Msg & Data rates may apply). You can reach our support team at support@axlehire.com or 1-855-249-7447."*

**No active shipments** → show **"AxleHire"** name, with email/phone from the Consul config.

## 3. Active vs Inactive shipment statuses

"Active shipment" is determined by shipment status:

| Active | Inactive |
|---|---|
| `GEOCODED` | `RETURNED_TO_CLIENT_PENDING` |
| `RESCHEDULED` | `RETURNED_TO_CLIENT_SUCCEEDED` |
| `ASSIGNED` | `DROPOFF_SUCCEEDED` |
| `PICKUP_SUCCEEDED` | `DISPOSABLE` |
| `CREATED` | `UNSERVICEABLE` |
| | `UNDELIVERABLE` |
| | `CANCELLED_BEFORE_PICKUP` |
| | `CANCELLED_AFTER_PICKUP` |
| | `RETURN_FAILED` |
| | `DROPOFF_FAILED` |
| | `RETURN_SUCCEEDED` |
| | `PICKUP_FAILED` |
| | `GEOCODE_FAILED` |
| | `INVALID_ADDRESS` |

## 4. How to send START / STOP messages (testing / manual trigger)

To simulate an inbound START/STOP without a real phone (e.g. on staging where the SMS provider is a dummy), publish a `CommunicationMessage` to the **`incoming_sms`** queue (exchange `INCOMING_COMMUNICATION`). `IncomingCommunicationViaSMSWorker.replyStandardCTIA()` processes it exactly as a real inbound SMS and updates the `unsubscribers` collection.

**Message body:**

```json
{
  "ts": "2026-06-18T09:13:09.228Z",
  "sender": "+14155550102",
  "recipients": ["+14155550199"],
  "body": "START",
  "recipient_type": "recipient",
  "sms_provider": "bandwidth"
}
```

| Field | Value |
|---|---|
| `body` | `STOP` to unsubscribe, `START` to re-subscribe. Must be exact (case-insensitive, trimmed). |
| `sender` | The recipient phone to unsubscribe / re-subscribe. The handler keys `unsubscribers` by `formatPhoneNumber(sender)` (`+1` + last 10 digits). **Do NOT** use the delivery number here. |
| `recipients` | The delivery number; only needs to be non-empty to pass validation. |

**Effect on the `unsubscribers` collection:**

- `STOP` → creates 3 records for the contact: `SMS`, `SMS_SERVICE`, `SMS_MARKETING` (`ineligible: false`).
- `START` → deletes those records (full re-subscribe).
- Texting `START` clears **all 3** types, whereas opting in through the tracking page (`DeliveryManager`) only clears `SMS` + `SMS_SERVICE` and keeps `SMS_MARKETING`.

**Verify:** check the `unsubscribers` collection, or the worker log on Datadog for the line `Standard CTIA <sender> <body>`.

## 5. SMS unsubscribe types

The `unsubscribers` collection uses a `type` field (`PreferenceType` enum) to scope what is blocked:

| Type | Meaning | Blocks |
|---|---|---|
| `SMS` | Global STOP (user texts STOP) | All SMS — both service and marketing |
| `SMS_SERVICE` | Unsubscribe service / transactional SMS | Service SMS only (delivery updates, tracking, notifications) |
| `SMS_MARKETING` | Unsubscribe marketing SMS | Marketing / promotional SMS only |

**Send-time gate** (`isPromotional` flag on the message):

- Service SMS (`isPromotional=false`) → blocked if `SMS_SERVICE` or `SMS` exists.
- Marketing SMS (`isPromotional=true`) → blocked if `SMS_MARKETING` or `SMS` exists.
- A global `SMS` record blocks both.

> A recipient opting in for delivery updates (service) does **not** automatically consent to marketing — that is why tracking-page opt-in clears `SMS` + `SMS_SERVICE` but keeps `SMS_MARKETING`.

## 6. The `ineligible` field

| Value | When | Behaviour |
|---|---|---|
| `true` | Unsubscribe caused by driver QUIT / SUSPENDED (system-enforced) | Automatically removed when the driver is reactivated |
| `false` | User-initiated unsubscribe (texted STOP, or opt-out) | Not auto-removed — requires START / opt-in to clear |

> SMS opt-in (IMPL-194) STOP records are always `ineligible: false`.

## 7. Send-time gate — exact implementation

Sections 5–6 describe the `unsubscribers` model conceptually. This section documents **where and how** the gate is actually enforced at send time, code-traced in `worker`.

### 7.1 Model — `CommunicationUnsubscriber` (collection `unsubscribers`)

| Field | Meaning |
|---|---|
| `type` | `SMS` (full opt-out) / `SMS_SERVICE` / `SMS_MARKETING` / `email` / `phone_call` |
| `contact` | Phone number (or email) — **already formatted** as stored |
| `unsubscribe_ts` | Timestamp of unsubscribe |
| `ineligible` | See §6 — **not read** by the send-time gate itself, descriptive only |

### 7.2 Gate — `TwilioSMSProviderWorker` (lines 181–197)

```java
List<String> sendable = message.recipients;
if (message.isForceSent) {                    // isForceSent = true -> bypasses unsubscribe check entirely
    ...
} else {
    List<String> unsubscribed = message.recipients.parallelStream()
        .filter(r -> hasUnsubscribed(formatPhoneNumber(r), message.isPromotional))
        .collect(...);
    sendable = recipients minus unsubscribed;
}
if (sendable.isEmpty()) return;               // nobody left -> quit, no send
```

- Phone numbers are formatted (`CommunicationViaSMSWorker.formatPhoneNumber`) **before** being matched against `contact`.
- `message.isForceSent == true` skips the unsubscribe check completely — used for messages that must always go out regardless of opt-out state.

### 7.3 Core logic — `hasUnsubscribed(phone, isPromotional)` (lines 553–583)

```java
// 1. Query unsubscribers by contact=phone, type in {SMS, SMS_SERVICE, SMS_MARKETING}
rows = communicationUnsubscriberDAO.listByContactsAndTypes([phone], [SMS, SMS_SERVICE, SMS_MARKETING]);
// 2. Promotional SMS + a SMS_MARKETING row -> block
if (isPromotional == true  && rows.anyMatch(type == SMS_MARKETING)) return true;
// 3. Service/notification SMS + a SMS_SERVICE row -> block
if (isPromotional != true  && rows.anyMatch(type == SMS_SERVICE))   return true;
// 4. A SMS row (recipient texted STOP) -> block ALL SMS
if (rows.anyMatch(type == SMS)) return true;
return false;   // not unsubscribed -> send allowed
```

This is the same three-tier logic as §5, now traced to its exact query and gate function.

### 7.4 Known risks

- `[RISK]` **Phone-format mismatch.** If the number's stored format in `unsubscribers.contact` differs from what `formatPhoneNumber()` produces at send time, the match **fails open** — i.e. the recipient is **not** suppressed even though they opted out. Worth a data-quality check when debugging a "why did an unsubscribed recipient still get texted" report.
- `[RISK]` **False `SENT` status on suppression.** The unsubscribe gate runs *after* an `sms_queue` row has already been created/queued (for queue-pipeline SMS types) — suppressing here blocks the actual message, but does not prevent the row from being marked as if it sent. Confirmed as a known issue on the rolled-shipment flow specifically — see [Shipment Rolled Notification §8](shipment-rolled-notification.md#8-known-issues--risks) — the same risk applies to any SMS type routed through `sms_queue` whose send is blocked at this gate, not just that one flow.
- `message.isForceSent` bypass is a deliberate escape hatch, not a bug — but worth checking which SMS types set it when investigating an unexpected send to an unsubscribed recipient.

### 7.5 Net effect

`unsubscribers` suppression is a **last-line check inside the send worker**, after any `sms_queue` row has already been created/queued. It does not prevent the row from existing — only the actual message from going out.
