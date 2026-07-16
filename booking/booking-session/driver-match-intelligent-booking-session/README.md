# Driver match - Intelligent Booking Session

> Source: [Confluence — Driver match - Intelligent Booking Session](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/999292972/Driver+match+-+Intelligent+Booking+Session)

This document provides the main configuration for an Intelligent Booking Session.

**Definition:** an Intelligent Booking Session is a booking session with a `booking_model` that allows matched drivers the flexibility to book tickets earlier than `booking_start_time` based on various criteria (such as the driver's home address, booking history, delivery preference).

A Booking Session is considered an Intelligent Booking Session when it satisfies **all** of the following conditions:

## 1. Booking session has a [Booking model](booking-model-configuration.md)

- Having `booking_session.attributes.booking_model_id` OR `booking_session.model_id`.
- To create a Booking session with a Booking model, use a `booking_session_template` with param `booking_model`.
- `booking_model_id` / `model_id` / `booking_session_template.booking_model` is taken from `booking_model.id` (MongoDB collection).

## 2. Driver belongs to `active_pools`

Driver belongs to the active pools in the `booking_model.active_pools` settings.

## 3. (Outdated) Global item_metadata setting

Having settings (check on MongoDB collection `item_metadata`: `{owner: "GLOBAL_BOOKING-MODEL"}`) following the priority to check: **Driver (DR) → crew → region → global**

- If `DR_id` has value = true → DR is applied for Intelligent BS; else → this DR is not enabled for the new settings.
- If `DR_id` is not listed → check the `crew` field: if any crew of the DR is listed here with value = true → DR belonging to this crew is applied for Intelligent BS; else these DRs are not applied.
- If `DR_id` is not listed and no crew of the DR is listed → check for Region → following the rule as above.
- With Global → apply for all if value = true.

**Note:**

- If a DR belongs to multiple crews (same region or different region) and one of them has value = true → the DR is enabled for Intelligent BS within all their regions.
- When a DR is not listed and belongs to several crews → DO NOT set up their crews inconsistently (some enabled, some disabled).

## Child pages

- [Booking model configuration](booking-model-configuration.md)
- [Driver preferred zone (Outdated)](driver-preferred-zone-outdated.md)
- [Notification for matched drivers](notification-for-matched-drivers.md)
- [Driver match on Driver App](driver-match-on-driver-app.md)
- [Dispatch App - Booking Detail popup](dispatch-app-booking-detail-popup.md)
- [Driver match - Caches](driver-match-caches.md)
- [Driver match score](driver-match-score.md)
