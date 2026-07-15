# Driver match - Caches

> Source: [Confluence — Driver match - Caches](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1048641645/Driver+match+-+Caches)

This document lists all the caches involved in updating data for the driver match.

→ Please clear the cache on Redis for these keys every time you directly update data in the Database.

| Cache type | Redis Cache key | Updated Cache | Example | TTL | Action related to Cache |
| --- | --- | --- | --- | --- | --- |
| Driver prefer zone | `{environment}-driver_prefer_zone_map` | | `staging-driver_prefer_zone_map` | 1800s | `getDriverPreferZoneByListDriverID`, `getDriverPreferZoneByListDriverCrewId` |
| Driver crew of driver (driver app use) | `{environment}-driver_crew_map_by_driver_id` | changed to use local cache, cannot be read anywhere | `staging-driver_crew_map_by_driver_id` | 1800s | `getDriverInfoByDriverId` |
| Driver id in a driver crew (Dispatch use) | `{environment}-driver_crew_map_by_driver_crew_id` | | `staging-driver_crew_map_by_driver_crew_id` | 1800s | `getDriverPreferZoneByListDriverCrewId` |
| Booking model | `{environment}-BM_{booking_model_id}` | | `staging-BM_booking_model_driver_match` | 300s | Get Booking list, detail, book/unbook |
| Number of open tickets | `{environment}-BS_{booking_session_id}:TB_{ticket_book_id}:NUM_TICKET_LEFT` | | `staging-BS_06e54186-...:TB_2899acf5-...:NUM_TICKET_LEFT` | 300s | Refresh BS, book/unbook |
| Booking details | `{environment}-BS_{booking_session_id}:detail` | | `staging-BS_06e54186-...:detail` | 300s | Open Booking detail on Driver app |
