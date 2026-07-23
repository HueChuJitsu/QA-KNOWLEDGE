# Notification for matched drivers

> Source: [Confluence — Notification for matched drivers](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1011777541/Notification+for+matched+drivers)

This document provides information about the flow of pushing notifications to matched drivers for an Intelligent Booking Session.

## 1. Configuration for Notification

`booking_session_template.announce_via_fcm = true`

Format of notification:

- Waves other than the final wave = `booking_model.waves.announcement_template`
- Final wave = `booking_session_template.announce_fcm_format`

## 2. Logic of pushing notification

- **API get matched drivers:** returns all drivers belonging to the region and in `active_pools` who have a `driver_prefer_zone` record and have an opened ticket for the prefer_zone in the current wave_time.

→ Worker Ticket Refresh triggers Worker BookingAnnouncement to push the notification to the list of matched drivers.

→ Then, around 30 mins (`booking_model.notification_boost_time_in_seconds`: 1800) prior to the `booking_start_time` of each wave (from API get matched drivers), [Worker Booking Announcement] is triggered and pushes the Intelligent Booking notification to all matched drivers after filtering (all at once):

- The `booking_start_time` for each driver belonging to `booking_model.active_pools` is displayed accordingly in the notification.

→ After notifications are sent to all matched drivers of a wave:

```
booking_session.attributes.sent_announcement_wave_1: true
booking_session.attributes.sent_announcement_wave_2: true
booking_session.attributes.sent_announcement_wave_3: true
```

If there are any updates (e.g. adding new drivers to `active_pools`) → new drivers will not receive notifications for this wave and will receive notifications in an upcoming wave.
