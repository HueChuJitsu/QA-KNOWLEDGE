# Nespresso Recycling — SMS Notifications

> **Project:** Nespresso Recycling · **Space:** Confluence — Engineering (ENG)
> **Part of:** [Recycling Feature Overview](recycling-feature-overview.md) · Last synthesized: July 2026
>
> SMS notifications sent to the recipient as part of the recycling ("pickup return")
> flow: a **pickup reminder** before pickup, and a **feedback request** after pickup.

## 1. Description

Two recipient-facing SMS messages support the recycling flow:

| SMS | When | Message type / config |
|-----|------|------------------------|
| Pickup reminder | Before pickup — a recycling bag is scheduled for pickup | `INFORM_RECIPIENT_PICKUP_REMINDER` (`telephonies`) |
| Feedback request | After pickup — ask about the pickup experience | `INFORM_RECIPIENT_FEEDBACK_REQUEST` (`telephonies`) + `FEEDBACK_RESPONSE` (`sms_template`) |

## 2. Business Flow

### 2.1 Pickup reminder SMS

> Source: *Nespresso — Send SMS to Recipient to Remind*

- **Message type:** `INFORM_RECIPIENT_PICKUP_REMINDER` in the `telephonies` table.
- **Purpose:** notify recipients via SMS when a recycling bag is scheduled for pickup.
- **Enablement scope:** can be enabled **by default for all clients**, or for **specific
  clients only**.

**Conditions to send** (both must be true):
1. The client **has pickup return enabled** (checked via client configuration).
2. The shipment **contains a recycling bag** (from shipment metadata / item tags).

When both are met, the system **automatically sends** the SMS using the
`INFORM_RECIPIENT_PICKUP_REMINDER` template.

### 2.2 Post-pickup feedback request SMS

> Source: *Configuration — Send SMS to Nespresso recycling recipients to ask about their pickup experience*

- **Request:** `telephonies` table, message type `INFORM_RECIPIENT_FEEDBACK_REQUEST` —
  asks the recipient about their pickup experience.
- **Response:** after the recipient replies, the auto-response is configured in MongoDB
  collection `sms_template`, title `FEEDBACK_RESPONSE`.

## 3. Spec / Rules

| Item | Value |
|------|-------|
| Reminder message type | `INFORM_RECIPIENT_PICKUP_REMINDER` |
| Feedback request message type | `INFORM_RECIPIENT_FEEDBACK_REQUEST` |
| Feedback response template | `sms_template` collection, title `FEEDBACK_RESPONSE` |
| Reminder precondition 1 | Client has pickup return enabled |
| Reminder precondition 2 | Shipment contains a recycling bag |
| Reminder enablement | Default for all clients, or specific clients only |

## 4. QA / Test notes

- **Reminder — both conditions met:** client with pickup return enabled + shipment with a
  recycling bag → recipient receives the `INFORM_RECIPIENT_PICKUP_REMINDER` SMS.
- **Reminder — condition missing:** client without pickup return, or shipment without a
  recycling bag → no reminder SMS.
- **Reminder enablement scope:** verify default-all-clients vs specific-client-only config.
- **Feedback request:** after pickup, recipient receives the
  `INFORM_RECIPIENT_FEEDBACK_REQUEST` SMS.
- **Feedback response:** recipient replies → auto-response uses `sms_template` /
  `FEEDBACK_RESPONSE`.

## Sources

| # | Confluence page | Last modified |
|---|-----------------|---------------|
| 1 | [Nespresso — Send SMS to Recipient to Remind](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1650196512) | Jul 31, 2025 |
| 2 | [Configuration — Send SMS to Nespresso recycling recipients to ask about their pickup experience](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1828126743) | Nov 02, 2025 |

> **Note:** the source pages contain SMS sample screenshots not reproduced here — refer to
> the linked pages for the visuals.
