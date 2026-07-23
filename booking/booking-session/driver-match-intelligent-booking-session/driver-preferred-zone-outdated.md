# Driver preferred zone (Outdated - Prod no setting driver prefer zone in booking_model)

> Source: [Confluence — Driver preferred zone (Outdated)](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1010040860/Driver+preferred+zone+Outdated+-+Prod+no+setting+driver+prefer+zone+in+booking_model)

This document provides information about how to generate data for `driver_prefer_zone`, which includes data of driver prefer location; table `driver_prefer_zone` in MongoDB (**Worker Driver prefer zone**).

Once a record at `driver_prefer_zone` is created, it will not be deleted even if the driver is removed from that region.

**Update 16 Jul:**

- Worker `DRIVER_PREFER_ZONE` is scheduled on the [Dashboard](https://dashboard.staging.gojitsu.com/schedules/).
- Once the worker is triggered, it checks any driver who has an event within the list of topics to update prefer_zone ([Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/workers/schedule_driver_prefer_zone/attributes/list_topic_update_prefer_zone_home_address/edit)) → then updates `driver_prefer_zone` data for all 3 types if present.

## I. Generate data for type = HOME_ADDRESS

- When a new driver signs up on the Driver App and signing the contract is done → the driver is automatically added to a driver crew by a worker → then worker [driver prefer zone] runs to generate a record at the `driver_prefer_zone` Mongo collection (type = HOME_ADDRESS) corresponding to their crew.
- With existing drivers, whenever there is any update related to home address OR removing/adding the driver to other crews → worker [driver prefer zone] automatically runs to update data for `driver_prefer_zone` (HOME_ADDRESS) correspondingly.

How `Worker Event-processor` works:

1. Get `driver_id` and home address from the Drivers table (Postgres).
2. Get the list of driver Crews → get Regions from `driver_crew` table (Mongo).
3. Get the list of zones (main zone) of each region from `routing_zone` table (Mongo).
4. Map the home address with the list of zones (step 3) of each region to find the PreferZone following the **Configs about distance**.

→ generate data to `driver_prefer_zone`; `type = HOME_ADDRESS` table in MongoDB like:

```json
{
  "_id": { "$oid": "665ec7ba23407b63492be322" },
  "driver_id": { "$numberLong": "8307" },
  "region": "LAX",
  "zone_prefer_data": [
    {
      "type": "HOME_ADDRESS",
      "zone_prefer": {
        "Z2": { "$numberLong": "2" },
        "Z12": { "$numberLong": "2" },
        "Z4": { "$numberLong": "2" },
        "Z5": { "$numberLong": "2" },
        "Z8": { "$numberLong": "2" },
        "Z9": { "$numberLong": "1" }
      }
    }
  ]
}
```

- If a driver belongs to multiple crews with different regions → there will be multiple `driver_prefer_zone`s (same `driver_id`) corresponding to each region.
- **Configs about distance:** [Link config](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/workers/schedule_driver_prefer_zone/attributes/distance_max_for_set_zone_prefer/edit)
  - One for checking whether the polygon is adjacent if the home_address is in a certain zone.
    - The zone of the driver's home address has priority = 1.
    - Other zones within `distance_for_check_polygon_adjacent` from the zone of the home address have priority = 2.
  - One for checking if there is any zone within the radius (distance) from home_address if the home_address is **not** in a certain zone.
    - There is no zone with priority = 1.
    - All zones within `distance_max_for_set_zone_prefer` have priority = 2.

  Config in Consul:

  ```
  distance_for_check_polygon_adjacent=0.01
  distance_max_for_set_zone_prefer=0.5
  ```

  The unit is degree. One degree of latitude or one degree of longitude at the equator is 60 nautical miles or **111.12 kilometers**.

## II. Generate data for type = DRIVER_PREFERENCE

1. When a driver is created successfully (completes the questions section → passes quiz → fulfills account info → completes 100% personal information) → the worker automatically runs to generate `DRIVER_PREFERENCE` for `driver_prefer_zone`.
   - The worker gets data from MongoDB collection `preference_delivery.prefs.values` (type = `DROP_OFF`).
   - All zones have `priority_level = 1`.
2. Whenever a driver updates their prefer dropoff zone → the worker automatically runs to update data of `driver_prefer_zone.DRIVER_PREFERENCE` following the updated data.

```json
{
  "_id": { "$oid": "665ec7ba23407b63492be322" },
  "driver_id": { "$numberLong": "8307" },
  "region": "LAX",
  "zone_prefer_data": [
    {
      "type": "DRIVER_PREFERENCE",
      "zone_prefer": {
        "Z1": { "$numberLong": "1" },
        "Z4": { "$numberLong": "1" },
        "Z8": { "$numberLong": "1" }
      }
    }
  ]
}
```

## III. Generate data for type = BOOKING_HISTORY

The system generates `zone_prefer` data with type Booking_history from the Booking history of the Driver in each region.

1. A schedule called `Driver prefer zone update` runs once a day to create/update `zone_prefer` data with type Booking_history zone for each driver in each region.
2. How the Worker calculates Driver prefer priority based on their Booking history:

   ```
   Weight Zone priority calculation formula = sum [number of tickets of zone * decay_rate ^ day]
     (day = datetime run schedule - target datetime of BS of tickets, day is integer)
   Zone priority = round [Max (Weight Zone priority) / Weight Zone priority * 100]
   ```

   ex: Schedule runs on 30 May at 10 pm GMT and attributes are:

   ```json
   {
     "regions": "LAX",
     "number_days": "10",
     "decay_rate": "0.95"
   }
   ```

   → Worker gets the list of Booking sessions with target date-time within (30 May 10pm − 10 days = 20 May 10pm) in region LAX → gets the list of tickets in `booking_session.groups.items` → gets the list of Claimed and Completed tickets, categorizes them by Driver, then calculates the Driver prefer zone by Booking history.

   (as BS target date-time in LAX is normally dd-mm-yyyy 19:00:00, the Worker gets Booking sessions from 30 May to 20 May)

   - Z3 Weight Zone priority = 2·0.95^1 + 4·0.95^2 + 4·0.95^3 + 2·0.95^8 + 3·0.95^9 + 2·0.95^10
   - Z2 Weight Zone priority = 2·0.95^3 + 1·0.95^10
   - → Z3 priority = 100, Z2 priority = 577

3. If the Driver has no **Claimed** or **Completed** ticket in the date range the Worker checks, the existing Driver prefer zone type Booking history is not updated.
4. The Booking service then maps the Booking history priority value to respective values 1, 2, 3, … to match the Booking model setting. The smallest Booking history priority value of a Driver is mapped to 1, and so on with other values.

## Update 12/2024: Ticket link [ALT-248](https://gojitsu.atlassian.net/browse/ALT-248)

- Add one more flag `update_preference_for_whole_region`: if true, update driver preferences for whole regions; else update based on events within (firing time − 1 day) and firing time.
- Add one more flag `allow_no_preference_order`: if true, save a zone that has no `preferred_order` as priority order = 1 (`preference_delivery.prefs.values.preferred_order` Mongo collection); if false, do not get a zone with no prefer_order in `driver_prefer_zone`.
- `is_data_migration_required: "true"` to allow updating home_address of all drivers for whole regions.
