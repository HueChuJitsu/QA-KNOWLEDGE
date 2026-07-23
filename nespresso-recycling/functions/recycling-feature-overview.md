# Nespresso Recycling — Feature Overview

> **Project:** Nespresso Recycling · **Space:** Confluence — Engineering (ENG)
> **Synthesized from 5 Confluence pages** (see [Sources](#sources)) · Last synthesized: July 2026
>
> This document consolidates the configuration and business logic for the Nespresso
> recycling ("pickup return") feature across the Driver App, Recipient Page, and SMS
> notifications.

## 1. Overview

The recycling feature lets a driver **pick up an empty recycling bag** ("pickup return")
from the recipient when delivering a Nespresso shipment. A shipment participates in
recycling when it is flagged `pickup_return = true`; the system then generates a
**recycling assignment, package, and stop** when the driver arrives.

The feature spans four areas:

| Area | What it covers | Section |
|------|----------------|---------|
| Enablement & config | Turn recycling on per client / region / warehouse; labels, bonus verbiage | [§2](#2-enabling--configuration) |
| Driver App validation | Barcode / QR validation pop-up when scanning the bag | [§3](#3-driver-app--barcode-validation) |
| Recipient Page | Show pickup-return status to the recipient | [§4](#4-recipient-page--pickup-return-status) |
| SMS notifications | Pickup reminder + post-pickup feedback request | [recycling-sms-notifications.md](recycling-sms-notifications.md) |

---

## 2. Enabling & Configuration

> Source: *Enabling Recycling Feature in Driver App: Configuration and Setup Guide*

### 2.1 How to enable

A shipment must have `pickup_return = true`. Recycling assignment / package / stop are
then generated when the driver arrives.

Enable per **region** and **warehouse** for a client under **client setting → delivery
setting**, using the `enable_recycling` key:

```
"enable_recycling": "RG_SFO,WH_43"
```

- Format: `RG_<region>,WH_<warehouse>` (e.g. region SFO + warehouse 43).
- Config is saved to MongoDB `client_settings`.

### 2.2 Assignment / stop labels (Consul)

Configured in Consul under `prod/apps/driverappapi/`:

| Consul key | Purpose |
|------------|---------|
| `recycle_assignment_label` | Assignment label — format `Pickup Return <{assignment_label}> <{predicted_start_ts}>` |
| `recycle_assignment_description` | Assignment description |
| `recycle_stop_description` | Stop description |

### 2.3 Pickup-return bonus verbiage (Consul)

Under `staging/apps/driverappapi/` (staging paths shown):

| Consul key | Applies to |
|------------|-----------|
| `ic_recycling_bonus_note` | IC drivers |
| `dsp_recycling_bonus_note` | DSP drivers |
| `recycling_bonus_info` | Explanation verbiage (shown when driver taps for info) |

### 2.4 Where the verbiage shows

**Direct booking**

| Client Settings | Schedule Settings |
|-----------------|-------------------|
| Dashboard app → Clients → enter clientID → **Edit** → `enable_recycling` tab → set enabled regions. Saved to MongoDB `client_settings`. | Insert a record into MongoDB `schedule` (see fields below). |

`schedule` record fields:

| Field | Meaning |
|-------|---------|
| `client_ids` | Clients for which recycling is enabled |
| `regions` | Regions where recycling is enabled. **Blank = all regions** |
| `rate` | Amount received after successfully returning a bag |
| `driver_types` | Enabled driver types (`DSP`, `IC`, `PLATFORM`). Blank = not restricted |
| `courier_ids` | For `PLATFORM` / `DSP`: courier IDs enabled. **Blank = all couriers** |
| `driver_ids` | For `IC`: driver IDs enabled. **Blank = all drivers** |
| `type` | `"surcharge_payment"` |
| `name` | `"Recycle Payment"` |
| `active` | `true` |

**Booking session**

If at least one `client_id` is enabled in the delivery setting on the dashboard, the
pickup return is shown on the booking session.

---

## 3. Driver App — Barcode Validation

> Source: *Validate pop-up in Drive App* · Related tickets: [MOB-1083](https://gojitsu.atlassian.net/browse/MOB-1083), [MOB-1084](https://gojitsu.atlassian.net/browse/MOB-1084)

### 3.1 Valid barcode format

- Consul key: `delivery_recycling_regex` — default `^1Z.{18}$`
  (`staging/apps/driverappapi/mobile_app_config/delivery_recycling_regex`)
- A valid barcode **must start with `1Z`** and be **exactly 18 characters** long.

### 3.2 Invalid barcode handling

If the scanned barcode does **not** start with `1Z`, **or** is **not exactly 18
characters**, the system triggers the **Invalid Barcode Scanned** pop-up.

- Patterns to ignore are configured in Consul:
  `mobile_app_config/delivery_recycling_patterns_to_be_ignored`

---

## 4. Recipient Page — Pickup Return Status

> Source: *Recipient Page — Show pickup return status in Recipient page*

The recipient page displays the pickup-return outcome using Consul display templates
under `prod/apps/recipient_api/display_template/`:

| Status | Display template (Consul) |
|--------|---------------------------|
| `PICKUP_COMPLETED` | `RECYCLE:GATHER_COMPLETED` |
| `PICKUP_FAILED` | `RECYCLE:GATHER_FAILED` (message content updated 27 Sep) |

---

## 5. SMS Notifications

> **Moved to a dedicated doc:** [recycling-sms-notifications.md](recycling-sms-notifications.md)

Recipient-facing SMS for the recycling flow:

| SMS | Message type | Config |
|-----|--------------|--------|
| Pickup reminder | `INFORM_RECIPIENT_PICKUP_REMINDER` | `telephonies` table |
| Feedback request | `INFORM_RECIPIENT_FEEDBACK_REQUEST` | `telephonies` + `sms_template` / `FEEDBACK_RESPONSE` |

See [recycling-sms-notifications.md](recycling-sms-notifications.md) for send conditions,
enablement scope, and QA notes.

---

## Sources

| # | Confluence page | Last modified |
|---|-----------------|---------------|
| 1 | [Enabling Recycling Feature in Driver App: Configuration and Setup Guide](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1063452688) | Sep 15, 2025 |
| 2 | [Nespresso — Send SMS to Recipient to Remind](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1650196512) | Jul 31, 2025 |
| 3 | [Recipient Page — Show pickup return status in Recipient page](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1683390483) | May 07, 2025 |
| 4 | [Validate pop-up in Drive App](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1687683076) | May 09, 2025 |
| 5 | [Configuration — Send SMS to Nespresso recycling recipients to ask about their pickup experience](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1828126743) | Nov 02, 2025 |

> **Note:** the source pages contain screenshots (config UIs, verbiage examples, SMS
> samples) that are not reproduced here. Refer to the linked pages for the visuals.
