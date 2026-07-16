# Driver match on Driver App

> Source: [Confluence — Driver match on Driver App](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1009844303/Driver+match+on+Driver+App)

This document provides information about the display of Booking Session / Booking group details in the Driver App.

## I. Displaying of BS

1. If in a wave the total number of open tickets for a preferZone = 0 because of one of these reasons:
   - `max_reservation = 0`
   - the Zone has 0 tickets when created
   - `num_ticket_ratio * total number of tickets for Zone < 0.5`

   → Booking session will NOT appear in the Booking screen.

2. If in a wave the total number of open tickets for a preferZone = 0 because:
   - the Zone has open tickets (`num_ticket_ratio * total number of tickets for Zone >= 0.5`) but they have already been booked by other drivers

   → Booking session will appear in the Booking screen with '0 available'.

3. If the driver's preferZone has a Lite_route and both the Normal Zone and ZLite have open tickets for a wave → the Lite Zone is counted as '1 available'.

## II. Displaying of Booking group details screen

1. `Limit` value:
   - Booking session max reservation is based on `booking_session_template.reservation`.
   - Real booking session max reservation is the min of (Booking session max reservation, Global booking limit, Global booking limit new driver, Booking session wave limit, and Driver probation).
2. `Booking@` time:
   - Should be the same as `booking_time` at the Booking tab.
   - Based on `booking_model.active_pools.boost_time + booking_model.booking_start_time_in_advance` of each wave.
   - In case there is a gap time between waves → during the 30 mins (`booking_model.notification_boost_time_in_seconds`: 1800) before the `booking_start_time` of each driver → `Booking@` time is shown for the upcoming wave, but the driver can't book until their `booking_time` (e.g. 30 mins before wave 2 → booking time of BS = booking time of wave 2).
