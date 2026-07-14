# Send message for ticket booking

> Source: [Confluence — Send message for ticket booking](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/976814100/Send+message+for+ticket+booking)

On the booking session screen there are 3 places for dispatch to send a message to a driver.

A recent change added a new config: `ticket_booking_push_notification`, which can be set up by region and globally. The screen behaves differently across three states:

- **Old version**
- **New version with config ON**
- **New version with config OFF** (default: Send SMS)

The 3 message entry points on the booking session screen each render in each of the three states above (see screenshots in Confluence).

## How the app receives the notification

| Case | Behavior |
| --- | --- |
| Send message to all drivers **without** HTML | When the driver clicks **Got It**, just close the popup. |
| Send message to all drivers **with** HTML | When the driver clicks **Got It**, close the popup. |
| Push notification to drivers with a ticket in a specific zone | Same title as the case sent to a specific driver; will push a notification to all drivers who booked a ticket of the zone. |
| Push notification to a specific driver | Notification is pushed to that driver only. |
