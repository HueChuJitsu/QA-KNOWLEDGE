# IMPL-194 — Individual Recipient SMS Opt-In (Feature Documentation)

## Overview

Recipients can opt into SMS (and optionally voice-call) delivery updates directly from the package
tracking page. They enter a phone number, verify their identity against a configured shipment field, and
consent. Once consented, the recipient-confirmed phone takes precedence over the manifest phone for all
delivery-notification SMS.

**Phase 1 is shipment-scoped:** consent is stored in `PhoneOptInConsent` keyed by `shipment_id`. No
`CustomerProfile` mutation, no cross-shipment propagation. Profile-wide opt-in deferred to L2/L3.

| Attribute | Value |
| --- | --- |
| Services | `recipient-api`, `recipient-web`, `worker` |
| Data model | `PhoneOptInConsent` (Mongo collection `phone_opt_in_consent`, key `shipment_id`) |
| Tickets | IMPL-194 · follow-ups IMPL-213, IMPL-214 |

## Architecture & Components

| Repo / Service | Component | Role |
| --- | --- | --- |
| `model` | `PhoneOptInConsent` | Consent entity (6 fields incl. audit IP + timestamp) |
| `dao` | `PhoneOptInConsentDAO` / Mongo impl | Shipment-keyed CRUD; reversible-encrypted phone, plaintext IP, UTC timestamps |
| `recipient-api` | `DeliveryManager`, `ShipmentFieldMatcher` | Verify + submit endpoints, rate limiting, identity matching |
| `recipient-web` | `SmsOptIn`, `SmsVerificationModal`, `DeliveryStore` | Tracking-page UI + actions |
| `worker` | `SMSComposer`, 7 SMS workers, `SMSOptInWelcomeHandler` | Send gate + phone resolution + CTIA welcome |

## Endpoints (recipient-api)

| Endpoint | Purpose |
| --- | --- |
| `POST /delivery/{tracking_code}/{shipment_id}/sms-opt-in/verify` | Identity verification; two-bucket Redis rate-limit; issues one-time verify token |
| `POST /delivery/{tracking_code}/{shipment_id}/sms-opt-in` | Submit consent; consume token; upsert `PhoneOptInConsent`; clear unsubscribers |

## Configuration

Standard Jitsu pattern: **Consul global default + per-client override via `ClientSettings.trackingSettings`**.
Resolution: `client override > Consul default > unset (fail-closed)`. Merged by
`DeliveryManager.getTrackingSettings(clientId)` with a **300-second TTL cache** on the Consul read.

| Key | Type | Default | Read by | Purpose |
| --- | --- | --- | --- | --- |
| `enable_recipient_sms_opt_in` | `"true"`/`"false"` | `"false"` (fail-closed) | recipient-api, recipient-web | Master gate. ≠`"true"` ⇒ section hidden, endpoints inert |
| `enable_recipient_call_opt_in` | `"true"`/`"false"` | `"false"` | recipient-api, recipient-web | Voice-call checkbox. Off ⇒ UI hides it + API coerces `call_opt_in=true → false` |
| `sms_opt_in_verification_field` | enum | unset ⇒ hidden | recipient-api, recipient-web | Field recipient must match to verify (see below) |
| `enforced_sms_enabled` | `"true"`/`"false"` | unset (≡false) | recipient-web | `"true"` ⇒ already enrolled ⇒ section hidden |
| `sms_opt_in_resubscribe_message` | free text | hardcoded FE default | recipient-web | Hint shown when `was_unsubscribed=true` |

Default resubscribe message (FE constant): *"If you previously stopped our messages, reply START to our last
delivery notification to re-activate updates."*

### Verification field mapping (`ShipmentFieldMatcher`)

`sms_opt_in_verification_field` allowed values and the shipment field each maps to:

```text
order_number   → shipment.orderNumber                    (case-insensitive, trimmed)
internal_id    → shipment.internalId                      (case-insensitive, trimmed)
zip            → shipment.dropoffAddress.zipcode          (ZIP+4 prefix-matched both ways; compares 5-digit prefix)
delivery_items → shipment.deliveryItems                   (case-insensitive, trimmed)
house_number   → first whitespace token of dropoffAddress.street
extra.<key>    → shipment.extra[key]
```

SMS opt-in uses `matchesTrimmedIgnoreCase`; POD verify reuses the same matcher with `matchesExact`
(case-sensitive).

### Consul keys (per environment)

```text
/<env>/apps/recipient_api/default_settings/enable_recipient_sms_opt_in   = "true"   # prod: "false" until QA
/<env>/apps/recipient_api/default_settings/enable_recipient_call_opt_in  = "false"
/<env>/apps/recipient_api/default_settings/sms_opt_in_verification_field = <value>

# Worker event-handler subscription (pod restart required)
/<env>/apps/workers/<namespace>/handlers/com.axlehire.worker.event.handler.shipment.SMSOptInWelcomeHandler
    → topic SHIPMENT.MODIFIER.sms-opt-in
```

## Data Model — `PhoneOptInConsent` (`phone_opt_in_consent`)

| Field | DB column | Notes |
| --- | --- | --- |
| `shipmentId` | `shipment_id` | Primary key (L1 shipment-scoped) |
| `optInPhoneNumber` | `opt_in_phone_number` | Reversible-encrypted |
| `smsOptIn` | `sms_opt_in` | SMS consent flag |
| `callOptIn` | `call_opt_in` | Voice-call consent flag |
| `optInTimestamp` | `opt_in_timestamp` | UTC, consent time |
| `optInIp` | `opt_in_ip` | Plaintext (TCPA audit; X-Forwarded-For leftmost token) |

## Feature Logic

### Section visibility (frontend)

```text
visible       = enable_recipient_sms_opt_in == "true"
             && sms_opt_in_verification_field is set
             && enforced_sms_enabled != "true"
call checkbox  = visible && enable_recipient_call_opt_in == "true"
```

### ZIP code masking in "Deliver to:" (privacy)

**New behavior (shipped with this feature):** the **zipcode** in the **Deliver to:** section is only
displayed **after the recipient has been authenticated** (identity verified). Before verification the
ZIP is hidden/masked, so it cannot be used to guess the `zip` verification value.

### Identity verification (`verify` endpoint)

- Matches input against `sms_opt_in_verification_field` via `ShipmentFieldMatcher.matchesTrimmedIgnoreCase`.
- On success: writes a single-use verify token (TTL 900s) and resets the per-(shipment, IP) rate bucket.

**Rate limiting** (hardcoded in `DeliveryManager.verifySmsOptInReference`):

| Bucket | Cap | Window | Reset |
| --- | --- | --- | --- |
| per (shipment, IP) | 5 | 30 min (sliding) | cleared on successful verify |
| per IP | 10 | 30 min (sliding) | never on success (cumulative) |

`checkRateLimit(key, 6L\|11L, 1800L)` — pre-increment; TTL refreshed on every call ⇒ **sliding window**.
Redis down ⇒ **503 fail-closed**. ⚠️ Not configurable. IP from `X-Forwarded-For` (leftmost), fallback
`getRemoteAddr()`.

### Consent submission (`sms-opt-in` endpoint)

1. Validate phone format (10–15 digits).
2. Atomically consume the one-time verify token.
3. Upsert `PhoneOptInConsent` (encrypted phone, flags, UTC timestamp, plaintext IP).
4. Clear `CommunicationUnsubscriber` for `SMS` + `SMS_SERVICE` (keeps `SMS_MARKETING`).
5. If `enable_recipient_call_opt_in != "true"` ⇒ coerce `callOptIn` to `false`.

### Opt-in phone resolution (`SMSComposer.resolveRecipientPhone`)

```text
if consent.smsOptIn == true && consent.optInPhoneNumber non-blank → opt-in phone
else                                                              → shipment.customer.phoneNumber (manifest)
```

Gate `isShipmentSmsEnabled`: `shipment.smsEnabled == true` **OR** consent `smsOptIn == true` — so opt-in
works even when the manifest flag is false. Resolution is **type-agnostic**: all recipient-notification SMS
prefer the opt-in phone.

### First-contact welcome (`SMSOptInWelcomeHandler`)

Subscribes to `SHIPMENT.MODIFIER.sms-opt-in`. Fires `CTIA_REPLY_START` only when `sms_opt_in=true`,
`was_unsubscribed=false`, `has_phone=true`. Re-subscribers do not receive the welcome again. Per-shipment
dedupe lock (5-min TTL).

### Unsubscribe types (`unsubscribers`, `PreferenceType`)

| Type | Scope | Blocks |
| --- | --- | --- |
| `SMS` | global STOP | all SMS |
| `SMS_SERVICE` | transactional | service SMS (delivery updates) |
| `SMS_MARKETING` | promotional | marketing SMS |

| Action | Effect |
| --- | --- |
| STOP (keyword) | creates `SMS` + `SMS_SERVICE` + `SMS_MARKETING` |
| Opt-in submit | clears `SMS` + `SMS_SERVICE`, keeps `SMS_MARKETING` |
| START (keyword) | clears all three |

`ineligible=true` ⇒ driver QUIT/SUSPENDED block (auto-removed on reactivation); `false` ⇒ user-initiated.
Carrier-level (Twilio/Bandwidth) STOP also requires texting START to truly re-enable — clearing the DB
record alone is not enough.

## Shipment Detail Log

After a recipient updates the opt-in phone, a log is written and shown in the **shipment detail**
activity feed in the form:

```text
Application [recipient] sms-opt-in Shipment [shipment_id]
```

Log content (`SHIPMENT.MODIFIER.sms-opt-in` event):

```json
{
  "id": "",
  "category": "SHIPMENT",
  "type": "MODIFIER",
  "action": "sms-opt-in",
  "ts": "{ts time when update the opt-in}",
  "source": {
    "attributes": {
      "version": "1.0.23"
    },
    "uid": "AP_recipient"
  },
  "object": {
    "uid": "SH_{shipment_id}"
  },
  "subject": {
    "uid": "AP_recipient"
  },
  "state": {
    "call_opt_in": "true/false",
    "has_phone": "true/false",
    "sms_opt_in": "true/false",
    "was_unsubscribed": "true/false"
  },
  "ephemeral": false
}
```

| Field | Notes |
| --- | --- |
| `action` | Always `sms-opt-in` |
| `ts` | Timestamp when the opt-in was updated |
| `object.uid` | `SH_` + `shipment_id` |
| `source.uid` / `subject.uid` | `AP_recipient` (event originates from the recipient app) |
| `state.call_opt_in` | Voice-call consent flag at update time |
| `state.has_phone` | Whether an opt-in phone is present |
| `state.sms_opt_in` | SMS consent flag at update time |
| `state.was_unsubscribed` | Whether the phone was previously unsubscribed |

This is the same `SHIPMENT.MODIFIER.sms-opt-in` event that `SMSOptInWelcomeHandler` subscribes to
(see [First-contact welcome](#first-contact-welcome-smsoptinwelcomehandler)).

## SMS Send Pipelines

All paths resolve opt-in phone via `SMSComposer.resolveRecipientPhone`.

| Pipeline | Path | SMS types | Timing |
| --- | --- | --- | --- |
| A — Immediate | `ShipmentStatusUpdater` (sms_trigger.yaml) → `SMSComposer` | Stop-status notifications (pickup/dropoff success/fail, coming-soon, next-in-line, about-delivery-pod, feedback, reminder) | Real-time |
| B — Queue | `SmsQueueWorker` → `sms_queue` → `SMSQueueProcessSchedule` | Access-code/address confirm (`CONFIRM_*`), DAS auto-corrected, reason-specific dropoff-failed (×7) | Scheduled per region |
| T — SmsTelephony | `RecipientSignWorker` / dataorch → `SmsTelephonyWorker` → `SMSComposer` | `SIGNATURE_TOKEN_SMS`, `ID_SCAN_SMS`, `TRACKING_LINK` | On-demand |
| — CTIA welcome | `SMSOptInWelcomeHandler` → `SMSComposer.triggerMessaging` | `CTIA_REPLY_START` | On opt-in |

### sms_queue (Pipeline B) lifecycle

- **Status:** `NOT_CONFIRMED` (enqueued, eligible) → `SENT`; `OPENED` (re-processed after 5 min); `IGNORED`
  (shipment DISPOSABLE); `CONFIRMED`.
- **Enqueue triggers** (`SmsQueueMessageType`): `FAILED_STOP`, `DRIVER_ACCEPT_ROUTE`, `INBOUND_SCAN`,
  `DELIVERABLE_ADDRESS_INCIDENT`.
- **Scheduled send** (`SMSQueueProcessSchedule`, per region): queries NOT_CONFIRMED (last 1 day) + OPENED
  (>5 min), filters `sentCount < 2` (max 2 sends), Redis lock per shipment, sends via `SMSComposer`.

## Acceptance Criteria (from ticket)

- Recipient can opt-in to SMS delivery updates when `enable_recipient_sms_opt_in=true`.
- Verification accepts any configured `sms_opt_in_verification_field` (incl. `zip` ZIP+4, `house_number`).
- Per-(shipment, IP) cap 5 / 30-min ⇒ 6th returns 429; per-IP cap 10 / 30-min ⇒ 11th returns 429.
- Successful verify resets only the per-(shipment, IP) bucket; per-IP stays cumulative.
- Submitting a previously unsubscribed phone clears `CommunicationUnsubscriber` for SMS+SMS_SERVICE,
  returns `was_unsubscribed: true`.
- First-contact opt-in (`was_unsubscribed=false`) triggers `CTIA_REPLY_START`.
- All subsequent recipient-notification SMS prefer the opt-in phone over the manifest phone.
- `PhoneOptInConsent` records include consent timestamp + client IP for TCPA audit.
- Voice-call checkbox hidden when `enable_recipient_call_opt_in != "true"`; `call_opt_in=true`
  server-coerced to `false`.
- Consul defaults flow through `getTrackingSettings`; per-client overrides win.

## Apply / Restart Checklist

| Service | Type | Restart? | Reason |
| --- | --- | --- | --- |
| recipient-api | API | Yes | new endpoints + `DeliveryManager` opt-in flow |
| worker (7 SMS workers) | Worker | Yes | `PhoneOptInConsentDAO` injected at startup |
| event-processor (`SMSOptInWelcomeHandler`) | Worker | Yes | new Consul subscription key |
| recipient-web | Frontend | Deploy | new `SmsOptIn` component |
| Consul key change only | — | No | 300s TTL — wait ~5 min |

## Phase 1 Intentional Omissions

- `Shipment.smsEnabled` not mutated (avoids coupling opt-in to manifest data model).
- `CustomerProfile` not written (L2 profile-wide opt-in deferred).
- Phone-keyed DAO methods unused (L1 is shipment-keyed only).

## Manual STOP / START trigger (testing on staging, dummy provider)

To simulate an inbound START/STOP without a real phone, publish a `CommunicationMessage` to queue
`incoming_sms` (exchange `INCOMING_COMMUNICATION`); `IncomingCommunicationViaSMSWorker.replyStandardCTIA()`
processes it as a real inbound SMS and updates the `unsubscribers` collection.

```json
{
  "sender": "+14155550112",
  "recipients": ["+14155550199"],
  "body": "START",
  "recipient_type": "recipient",
  "sms_provider": "bandwidth"
}
```

- `body`: `STOP` (create unsubscribers) or `START` (delete) — exact, case-insensitive, trimmed.
- `sender`: the recipient phone to unsubscribe/re-subscribe (handler keys on `formatPhoneNumber(sender)` =
  `+1` + last 10 digits). **Not** the delivery number.
- `recipients`: delivery number — only needs to be non-empty.

## UI Test Cases

### Section UI

| Test case | Expectation |
| --- | --- |
| SMS enabled, shipment not yet delivered | Section shown: textbox **Mobile Phone Number** (placeholder `e.g +1 (555) 123-4567`), checkbox **Text me with delivery updates**, button **Sign Up for Updates** (disabled) |
| SMS + call enabled | As above + checkbox **Call me with delivery issues** |
| `enforced_sms_enabled = true` | Section hidden |
| Shipment delivered | Section not shown |
| `enable_recipient_sms_opt_in = false` (client override) | Section hidden |
| ZIP in **Deliver to:** before recipient is authenticated | Zipcode hidden/masked |
| ZIP in **Deliver to:** after recipient is authenticated | Zipcode displayed |

### Phone validation

| Test case | Expectation |
| --- | --- |
| Input text / invalid phone → Sign Up | Error: *Please enter a valid phone number.* |
| Input valid phone | "Verify Your Identity" pop-up shown, matching `sms_opt_in_verification_field` |

### Verification & rate limit

| Test case | Expectation |
| --- | --- |
| Incorrect verification value | Error: *Unable to verify. Please check your connection and try again.* |
| 6th attempt (per shipment+IP) | Cannot get delivery updates (429) |
| 11th attempt per IP | Pop-up: *Too many incorrect attempts. Please contact support for assistance.* + Cancel |

### Verification field variants

| Setting | Prompt / behavior |
| --- | --- |
| `order_number` | "Order Number" textbox, case-insensitive |
| `internal_id` | "Reference Number" textbox |
| `delivery_items` | "Delivery Items" textbox |
| `zip` | "Delivery ZIP Code"; ZIP5 or ZIP+4 match on 5-digit prefix |
| `house_number` | First token of dropoff street |
| `extra.<key>` | Title-cased key; value at `shipment.extra[key]` |

### Unsubscribe / re-subscribe

| Test case | Expectation |
| --- | --- |
| Opt-in with previously unsubscribed phone | *You're signed up...* + resubscribe hint; `was_unsubscribed: true`; no CTIA welcome |
| Opt-in Call only (no SMS) | `CTIA_REPLY_START` NOT sent |

### SMS types routed to the opt-in phone

Once a recipient has opted in, **all** recipient-notification SMS prefer the opt-in phone over the
manifest phone (`SMSComposer.resolveRecipientPhone`). Each of the following is sent to the opt-in phone:

| Test case | Expectation |
| --- | --- |
| `INFORM_RECIPIENT_PICKUP_SUCCEEDED` sent when pickup succeeds | Sent to the opt-in phone |
| `INFORM_RECIPIENT_PICKUP_SUCCEEDED_SIGNATURE_REQUIRED` sent when pickup succeeds and `signatureRequired=true` | Sent to the opt-in phone |
| `INFORM_RECIPIENT_PICKUP_FAILED` sent when pickup fails | Sent to the opt-in phone |
| `INFORM_RECIPIENT_PICKUP_REMINDER` sent when the pickup reminder fires | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_SUCCEEDED` sent when delivery completes | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_SUCCEEDED_REASON` sent when delivery completes with a reason (left at door) | Sent to the opt-in phone |
| `INFORM_RECIPIENT_COMING_SOON` sent when driver is arriving soon | Sent to the opt-in phone |
| `INFORM_RECIPIENT_ABOUT_DELIVERY_POD` sent when POD/delivery info is shared | Sent to the opt-in phone |
| `INFORM_RECIPIENT_TRACKING_LINK` sent when a dispatcher sends the tracking link | Sent to the opt-in phone |
| `INFORM_RECIPIENT_FEEDBACK_REQUEST` sent after delivery | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED` sent when delivery fails (generic) | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_REASON` sent when delivery fails with a configured reason | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_UNRECOVERABLE` sent when delivery fails unrecoverably | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_NO_ACCESS_CODE` sent when delivery fails due to no access code | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_WRONG_ACCESS_CODE` sent when delivery fails due to wrong access code | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_ACCESS_BLOCKED` sent when delivery fails due to access blocked | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_ADDRESS_INVALID` sent when delivery fails due to invalid address | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_COMMUNICATION_ISSUE` sent when delivery fails due to a communication issue | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_ACCESS_DENIED_BY_DOORMAN` sent when delivery fails due to doorman denial | Sent to the opt-in phone |
| `INFORM_RECIPIENT_DROPOFF_FAILED_BUSINESS_CLOSED` sent when delivery fails due to business closed | Sent to the opt-in phone |
| `INFORM_RECIPIENT_SIGNATURE_TOKEN_SMS` sent when a signature link is requested | Sent to the opt-in phone |
| `INFORM_RECIPIENT_ID_SCAN_SMS` sent when an ID-scan link is requested | Sent to the opt-in phone |
| `CTIA_REPLY_START` sent on first-contact opt-in | Sent to the opt-in phone; **NOT** re-sent for re-subscribers |
