# Booking model configuration

> Source: [Confluence — Booking model configuration](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1009221708/Booking+model+configuration)

This document provides information about Booking Model settings.

```json
{
  "_id": { "$oid": "6628942123d6f48b4c282bec" },
  "type": "driver_match",
  "id": "booking_model_driver_match_trang_clone",
  "waves": [
    {
      "id": "wave_1",
      "booking_start_time_in_advance": "12h",
      "num_ticket_ratio": 0.1,
      "booking_end_time_in_advance": "8h",
      "max_reservation": 2,
      "priority_level": [1],
      "prefer_zone_types": ["HOME_ADDRESS"],
      "necessary_priority_level": [1],
      "necessary_zone_types": ["DRIVER_PREFERENCE"],
      "name": "Wave 1"
    },
    {
      "id": "wave_2",
      "name": "Wave 2",
      "booking_start_time_in_advance": "8h",
      "num_ticket_ratio": 0.2,
      "booking_end_time_in_advance": "0h",
      "max_reservation": 3,
      "priority_level": [1],
      "prefer_zone_types": ["BOOKING_HISTORY", "HOME_ADDRESS"],
      "necessary_priority_level": [1, 2, 3],
      "necessary_zone_types": ["DRIVER_PREFERENCE"]
    },
    {
      "id": "wave_3",
      "name": "Wave 3",
      "booking_start_time_in_advance": "0h",
      "num_ticket_ratio": 1,
      "max_reservation": 4,
      "is_final": true
    }
  ],
  "active_pools": [
    { "type": "POOL_1", "boost_time": 0 },
    { "type": "POOL_2", "boost_time": 900 },
    { "type": "POOL_3", "boost_time": -5400 },
    { "type": "POOL_4", "boost_time": -6300 }
  ],
  "notification_boost_time_in_seconds": 1800
}
```

- `priority_level` is applied for `prefer_zone_types`.
- `necessary_priority_level` is applied for `necessary_zone_types`.
- `necessary_zone_types`: if a Zone matches the `priority_level` and is in `prefer_zone_types` but is NOT listed in `necessary_zone_types` with `necessary_priority_level` → this Zone does not meet all the conditions to be displayed in the wave.
- `booking_start_time_in_advance`: the period of time prior to the `booking_time` of the BS.
- `booking_end_time_in_advance`: the number of hours prior to the `booking_time` of the BS.
- `num_ticket_ratio`: the ratio of total tickets that will be opened in the wave.
  - For example: group Z1 has 10 tickets when created AND wave_1 has `num_ticket_ratio = 0.1` → the open tickets for Z1 in wave_1 is 1 ticket.
- `max_reservation`: booking limit of a wave.
- `priority_level`: refers to the priority level in collection `driver_prefer_zone`.
  - E.g. DR_7972 has data at `driver_prefer_zone` → with type Home_address, Z1 has priority level = 1 and Z2 has priority level = 2 → in wave 1, Driver 7972 can book a ticket in zone Z1.
- `active_pools`: the pools the booking_model applies to.
- `notification_boost_time_in_seconds`: the time prior to `booking_start_time` of each wave that the BookingAnnouncement Worker is triggered to push the notification of Intelligent Booking to drivers.

## Updated 20/12/2024: Dynamic using formula

New collection: `driver_match_score`.

Data in `driver_match_score` is generated based on a formula in the respective Booking model and wave.

_ex:_

```json
"waves": [
    {
      "id": "wave_1",
      "booking_start_time_in_advance": "12h",
      "num_ticket_ratio": 0.1,
      "booking_end_time_in_advance": "8h",
      "max_reservation": 2,
      "name": "Wave Ngoc 1",
      "priority_level": [2],
      "prefer_zone_types": ["HOME_ADDRESS"],
      "announcement_template": "${num_ticket_ratio} tickets Tailored for you for ${session_name} will be available at ${booking_start_time}. Each driver can book ${max_reservation} ticket(s). There will be more tickets available at the standard booking session",
      "formula": "SELECT RG_LAX FILTER status IN (POWER, ACTIVE) PREFERED_ZONE (DRIVER_PREFERENCE(priority=1) UNION BOOKING_HISTORY(priority=1)) SCORING (OTD(weight=0.4, threshold>= 0.6, max_value=1.0) + Customer_Rating(weight=0.05, threshold>= 0.55, max_value=1.0) + Completion_Rate(weight=0.35, threshold>= 0.93, max_value=1.0) + OTP(weight=0.1, threshold>= 0.1, max_value=1.0) + OTC(weight=0.1, threshold>= 0.8, max_value=1.0)) TOP 1"
    },
    {
      "id": "wave_2",
      "booking_start_time_in_advance": "8h",
      "num_ticket_ratio": 0.2,
      "booking_end_time_in_advance": "0h",
      "max_reservation": 3,
      "name": "Wave Ngoc 2",
      "priority_level": [1],
      "prefer_zone_types": ["HOME_ADDRESS", "BOOKING_HISTORY"],
      "announcement_template": "${num_ticket_ratio} tickets Tailored for you for ${session_name} will be available at ${booking_start_time}. Each driver can book ${max_reservation} ticket(s). There will be more tickets available at the standard booking session",
      "formula": "SELECT RG_LAX FILTER status IN (POWER, ACTIVE, LESS_ACTIVE, ACTIVE_NEW) PREFERED_ZONE (DRIVER_PREFERENCE(priority=2) UNION BOOKING_HISTORY(priority=1) UNION HOME_ADDRESS(priority=1)) SCORING (OTP(weight=0.1, threshold>= 0.292, max_value=1.0) + OTD(weight=0.4, threshold>= 0.737, max_value=1.0) + Customer_Rating(weight=0.05, threshold>= 0.5, max_value=1.0) TOP 10"
    },
    {
      "id": "wave_3",
      "booking_start_time_in_advance": "0h",
      "max_reservation": 4,
      "num_ticket_ratio": 1,
      "name": "Wave Ngoc final",
      "is_final": true
    }
]
```

### How it's calculated

When the Schedule "Driver prefer zone" runs automatically or is triggered manually, the system uses the formula and respective data to calculate the driver's score and saves info of drivers that satisfy the formula in collection `driver_match_score`.

- **FILTER** status IN (POWER, ACTIVE, LESS_ACTIVE): get driver status from table `driver_tags.tag`.
- **PREFERED_ZONE** (DRIVER_PREFERENCE(priority=2) UNION HOME_ADDRESS(priority=2) UNION BOOKING_HISTORY(priority=2)): get driver prefer zone data from collection `driver_prefer_zone` (OUTDATED – no longer in use).
- **SCORING** (OTD(weight=0.5, threshold>= 0.992, max_value=1.0) + Customer_Rating(weight=0.05, threshold>= 0.50, max_value=1.0) + Completion_Rate(weight=0.45, threshold>= 0.95, max_value=1.0)): get on-time data; can check in the driver detail screen for each driver.

### Schedule Driver prefer zone — param clarification

- **update_score_only**: only get existing data and formula to calculate data in collection `driver_match_score`.
- **regions**: region that the schedule calculates for.
- **number_days**: related to Booking-history driver prefer zone, refer to [Driver preferred zone](driver-preferred-zone-outdated.md).
- **decay_rate**: related to Booking-history driver prefer zone, refer to [Driver preferred zone](driver-preferred-zone-outdated.md).
- **is_data_migration_required**: related to Driver-preference prefer zone.
- **update_preference_for_whole_region**: related to Driver-preference prefer zone. If true, re-calculate driver preference for all drivers in the region, else re-calculate driver preference for drivers with an "update driver preference" event in the current date.
- **allow_no_preference_order**: related to Driver-preference prefer zone. If true, re-calculate driver preference for drivers with no `order` param in dropoff zone info, else only re-calculate for drivers with an `order` param in dropoff zone info.

### Firing time

The system gets the list of booking sessions with target in 3 days from the firing date → calculate driver match score for the `booking_model` in those booking sessions.

### New value in `booking_model.type` = `driver_match_score`

- If type = `driver_match`, driver priority in each wave is based on the prefer zone and necessary zone settings in `booking_model` and the prefer zone type in collection `driver_prefer_zone`.
- If type = `driver_match_score`, driver priority in each wave is based on collection `driver_match_score`.
