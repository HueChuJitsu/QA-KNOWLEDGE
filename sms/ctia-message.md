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
