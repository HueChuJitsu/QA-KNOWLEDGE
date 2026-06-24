# Individual Recipient SMS Opt-In (IMPL-194) — Configuration & Logic Guide

Recipients opt into SMS (and optionally call) delivery updates from the tracking page after verifying
their identity against a configured shipment field. Once consented, the recipient-confirmed phone takes
precedence over the manifest phone for delivery-notification SMS. **Phase 1 is shipment-scoped** — consent
is stored in `PhoneOptInConsent` keyed by `shipment_id`; no `CustomerProfile` mutation, no cross-shipment
propagation.

## 1. Overview

| Attribute | Value |
| --- | --- |
| Feature | Individual Recipient SMS Opt-In (Phase 1) |
| Owner | ⚠️ TBD — confirm team (assignee: r.fox@gojitsu.com) |
| Services | `recipient-api`, `recipient-web`, `worker` |
| Data model | `PhoneOptInConsent` (collection `phone_opt_in_consent`, key `shipment_id`) |
| Tickets | IMPL-194 · follow-ups IMPL-213, IMPL-214 |

## 2. Architecture & Components

| Repo / Service | Component | Role |
| --- | --- | --- |
| `model` | `PhoneOptInConsent` | Consent entity (6 fields incl. audit IP + timestamp) |
| `dao` | `PhoneOptInConsentDAO` / Mongo impl | Shipment-keyed CRUD; reversible-encrypted phone, plaintext IP, UTC timestamps |
| `recipient-api` | `DeliveryManager`, `ShipmentFieldMatcher` | Verify + submit endpoints, rate limiting, identity matching |
| `recipient-web` | `SmsOptIn`, `SmsVerificationModal`, `DeliveryStore` | Tracking-page UI + actions |
| `worker` | `SMSComposer`, 7 SMS workers, `SMSOptInWelcomeHandler` | Send gate + phone resolution + CTIA welcome |

## 3. Configuration Reference

All settings follow the standard Jitsu pattern: **Consul global default + per-client override via
`ClientSettings.trackingSettings`**. Resolution: `client override > Consul default > unset (fail-closed)`.
Merged by `DeliveryManager.getTrackingSettings(clientId)` with a **300-second TTL cache** on the Consul read.

### 3.1 `enable_recipient_sms_opt_in` — master gate

| Field | Value |
| --- | --- |
| Type / Storage | `String "true"\|"false"` · Consul + client `trackingSettings` |
| Read by | `recipient-api` (`DeliveryManager`), `recipient-web` (`SmsOptIn`) |
| Default | `"false"` — **fail-closed** |
| Purpose | Master switch. ≠ `"true"` ⇒ opt-in section hidden, endpoints inert |

### 3.2 `enable_recipient_call_opt_in` — voice-call gate

| Field | Value |
| --- | --- |
| Type / Storage | `String "true"\|"false"` · Consul + client `trackingSettings` |
| Read by | `recipient-api`, `recipient-web` |
| Default | `"false"` (voice-call not yet shipped) |
| Purpose | Gates call checkbox. When off: UI hides it **and** API coerces `call_opt_in=true → false` server-side |

### 3.3 `sms_opt_in_verification_field` — identity field

| Field | Value |
| --- | --- |
| Type / Storage | `String` (enum) · Consul + client `trackingSettings` |
| Read by | `recipient-api` (`ShipmentFieldMatcher`), `recipient-web` (modal label) |
| Default | unset ⇒ section hidden |
| Allowed | `order_number` · `internal_id` · `zip` · `delivery_items` · `house_number` · `extra.<key>` |

`ShipmentFieldMatcher.extractExpectedValue(shipment, field)` mapping:

```text
order_number   → shipment.orderNumber
internal_id    → shipment.internalId
zip            → shipment.dropoffAddress.zipcode      (ZIP+4 prefix-matched both ways)
delivery_items → shipment.deliveryItems
house_number   → first whitespace token of dropoffAddress.street
extra.<key>    → shipment.extra[key]
```

### 3.4 `enforced_sms_enabled` — already-enrolled gate

| Field | Value |
| --- | --- |
| Type / Storage | `String "true"\|"false"` · Consul + client `trackingSettings` |
| Read by | `recipient-web` |
| Default | unset (≡ `"false"`) |
| Purpose | `"true"` ⇒ recipient already enrolled ⇒ section hidden |

### 3.5 `sms_opt_in_resubscribe_message` — resubscribe hint

| Field | Value |
| --- | --- |
| Type / Storage | `String` (free text) · client `trackingSettings` (frontend `settings`) |
| Read by | `recipient-web` only |
| Default | hardcoded FE constant: *"If you previously stopped our messages, reply START to our last delivery notification to re-activate updates."* |
| Notes | Shown only when `was_unsubscribed=true`; empty `""` ⇒ falls back to default |

## 4. Logic & Behavior

### 4.1 Section visibility (evaluated frontend)

```text
visible       = enable_recipient_sms_opt_in == "true"
             && sms_opt_in_verification_field is set
             && enforced_sms_enabled != "true"
call checkbox  = visible && enable_recipient_call_opt_in == "true"
```

### 4.2 Identity verification — `POST /delivery/{tracking_code}/{shipment_id}/sms-opt-in/verify`

- Matches `request.referenceValue` against `sms_opt_in_verification_field` via
  `ShipmentFieldMatcher.matchesTrimmedIgnoreCase` (trim + case-insensitive; `zip` is ZIP+4-aware).
- On success: writes a single-use verification token (`makeSmsVerifiedKey`, value 2, TTL 900s) and resets
  the per-(shipment, IP) rate bucket.

**Rate limiting** (hardcoded in `DeliveryManager.verifySmsOptInReference`):

| Bucket | Cap | Window | Reset |
| --- | --- | --- | --- |
| per (shipment, IP) | 5 | 30 min (sliding) | cleared on successful verify |
| per IP | 10 | 30 min (sliding) | never on success (cumulative) |

`checkRateLimit(key, 6L\|11L, 1800L)` — pre-increment; TTL refreshed on every call ⇒ **sliding window**
(block extends while attempts continue). Redis down ⇒ **503 fail-closed**. ⚠️ Not configurable.

### 4.3 Consent submission — `POST /delivery/{tracking_code}/{shipment_id}/sms-opt-in`

1. Validate phone format (10–15 digits) before any side effect.
2. Atomically consume the one-time verification token (`decrIfMoreThan`).
3. Upsert `PhoneOptInConsent` (`upsertByShipmentId`): encrypted phone, `smsOptIn`, `callOptIn`,
   `optInTimestamp` (UTC), `optInIp` (plaintext).
4. Clear `CommunicationUnsubscriber` records for `SMS` + `SMS_SERVICE` (keeps `SMS_MARKETING`).
5. `enable_recipient_call_opt_in != "true"` ⇒ coerce `callOptIn` to `false`.

### 4.4 Opt-in phone resolution — `SMSComposer.resolveRecipientPhone(shipment, consent)`

```text
if consent.smsOptIn == true && consent.optInPhoneNumber non-blank → opt-in phone
else                                                              → shipment.customer.phoneNumber (manifest)
```

`isShipmentSmsEnabled` gate: `shipment.smsEnabled == true` OR consent `smsOptIn == true`. The resolver is
**type-agnostic** — all recipient-notification SMS must prefer the opt-in phone.

> ⚠️ **Known gap (BUG):** SMS triggered from `data-orchestrator` (`AssignmentManager.sendSmsToClient`,
> line ~7129) hardcodes `shipment.customer.phoneNumber` and bypasses this resolution. Affects
> move-to-next-day, move-to-previous-day, and the dispatcher "Send SMS Notification" action — all send to
> the manifest phone. The opt-in resolution exists only in worker `SMSComposer`, not in data-orchestrator.

### 4.5 First-contact welcome — `SMSOptInWelcomeHandler`

Subscribes to `SHIPMENT.MODIFIER.sms-opt-in`. Fires `CTIA_REPLY_START` only when `sms_opt_in=true`,
`was_unsubscribed=false`, `has_phone=true` (gate). Re-subscribers do **not** receive the welcome again.
Per-shipment dedupe lock (5-min TTL).

### 4.6 Unsubscribe types (`unsubscribers` collection, `PreferenceType`)

| Type | Scope | Blocks |
| --- | --- | --- |
| `SMS` | global STOP | all SMS (service + marketing) |
| `SMS_SERVICE` | transactional | service SMS (delivery updates, etc.) |
| `SMS_MARKETING` | promotional | marketing SMS |

| Action | Effect on `unsubscribers` |
| --- | --- |
| STOP (keyword) | creates `SMS` + `SMS_SERVICE` + `SMS_MARKETING` |
| Opt-in submit (tracking page) | clears `SMS` + `SMS_SERVICE`, keeps `SMS_MARKETING` |
| START (keyword) | clears all three |

`ineligible=true` ⇒ unsubscribe from driver QUIT/SUSPENDED (auto-removed on reactivation);
`false` ⇒ user-initiated. Carrier-level (Twilio/Bandwidth) STOP also requires texting START to truly
re-enable delivery — clearing the DB record alone is not enough.

### 4.7 Manual STOP/START trigger (testing on staging, dummy provider)

Publish a `CommunicationMessage` to queue `incoming_sms` (exchange `INCOMING_COMMUNICATION`);
`IncomingCommunicationViaSMSWorker.replyStandardCTIA()` processes it as a real inbound SMS.

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

## 5. Data Model — `PhoneOptInConsent` (`phone_opt_in_consent`)

| Field | DB column | Notes |
| --- | --- | --- |
| `shipmentId` | `shipment_id` | Primary key (L1 shipment-scoped) |
| `optInPhoneNumber` | `opt_in_phone_number` | Reversible-encrypted |
| `smsOptIn` | `sms_opt_in` | SMS consent flag |
| `callOptIn` | `call_opt_in` | Voice-call consent flag |
| `optInTimestamp` | `opt_in_timestamp` | UTC, consent time |
| `optInIp` | `opt_in_ip` | Plaintext (TCPA audit) |

## 6. Apply / Restart Checklist

| Service | Type | Restart? | Reason |
| --- | --- | --- | --- |
| recipient-api | API | Yes | new endpoints + `DeliveryManager` opt-in flow |
| worker (7 SMS workers) | Worker | Yes | `PhoneOptInConsentDAO` injected at startup |
| event-processor (`SMSOptInWelcomeHandler`) | Worker | Yes | new Consul subscription key |
| recipient-web | Frontend | Deploy | new `SmsOptIn` component |
| Consul key change only | — | No | 300s TTL — wait ~5 min |

### Consul keys (per environment)

```text
/<env>/apps/recipient_api/default_settings/enable_recipient_sms_opt_in   = "true"   # prod: "false" until QA
/<env>/apps/recipient_api/default_settings/enable_recipient_call_opt_in  = "false"
/<env>/apps/recipient_api/default_settings/sms_opt_in_verification_field = <value>

# Worker event-handler subscription (pod restart required)
/<env>/apps/workers/<namespace>/handlers/com.axlehire.worker.event.handler.shipment.SMSOptInWelcomeHandler
    → topic SHIPMENT.MODIFIER.sms-opt-in
```

## 7. Test Cases

### 7.1 Section UI

| Test case | Expected result |
| --- | --- |
| SMS enabled, shipment not yet delivered | Section shown under progress tracking: textbox **Mobile Phone Number** (placeholder `e.g +1 (555) 123-4567`), checkbox **Text me with delivery updates**, button **Sign Up for Updates** (disabled) |
| SMS + call enabled, shipment not yet delivered | As above + checkbox **Call me with delivery issues** |
| `enforced_sms_enabled = true` | Section hidden |
| Shipment delivered | "Get Delivery Updates by SMS" section not shown |
| Shipment status `RETURN_SUCCEEDED` | Section shown |
| Shipment has no phone number | Section still shown |

### 7.2 Phone number validation

| Test case | Expected result |
| --- | --- |
| Input text into phone number → Sign Up | Error: *Please enter a valid phone number.* |
| Input invalid phone number → Sign Up | Error: *Please enter a valid phone number.* |
| Input valid phone number | "Verify Your Identity" pop-up shown, matching `sms_opt_in_verification_field` |

### 7.3 Identity verification & rate limit

| Test case | Expected result |
| --- | --- |
| Incorrect verification value | Error: *Unable to verify. Please check your connection and try again.* |
| More than 5 attempts (per shipment + IP) | Cannot get delivery updates |
| More than 10 attempts per IP | Pop-up "Verify Your Identity": *Too many incorrect attempts. Please contact support for assistance.* + **Cancel** button |

### 7.4 Verification field variants (`sms_opt_in_verification_field`)

| Setting | Expected "Verify Your Identity" UI / behavior |
| --- | --- |
| `order_number` | Prompt *Please enter your Order Number…*, Order Number textbox, **Cancel** & **Verify** |
| `internal_id` | Prompt *Please enter your Reference Number…*, Reference Number textbox, **Cancel** & **Verify** |
| `delivery_items` | Prompt *Please enter your Delivery Items…*, Delivery Items textbox, **Cancel** & **Verify** |
| `zip` (UI) | Prompt *Please enter your Delivery ZIP Code…*, Delivery ZIP Code textbox, **Cancel** & **Verify** |
| `zip` — input ZIP5 | Can update the opt-in phone |
| `zip` — input ZIP4 | Error: *The reference you entered does not match. Please try again.* |
| `zip` — input ZIP9 | Section still shown |
| `extra.<key>` (e.g. `job_id`) | ⚠️ TBD — confirm prompt/label/behavior |

### 7.5 Unsubscribe / re-subscribe & CTIA

| Test case | Expected result |
| --- | --- |
| Opt-in with a previously unsubscribed phone | *You're signed up for delivery updates!* + *If you previously stopped our messages, reply START to our last delivery notification to re-activate updates.* |
| Shipment-detail log when phone unsubscribed | Log records `was_unsubscribed: true` |
| Opt-in for **Call only** (no SMS) | `CTIA_REPLY_START` is **not** sent |

