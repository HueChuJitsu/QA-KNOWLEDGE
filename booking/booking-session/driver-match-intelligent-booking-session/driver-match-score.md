# Driver match score

> Source: [Confluence — Driver match score](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1446346893/Driver+match+score)

This document is about collection `driver_match_score`, which is used in Driver match phase 3 when we set up `booking_model.type = driver_match_score`.

Data in `driver_match_score` is generated based on a formula in the respective Booking model and wave.

_ex:_

```
SELECT RG_LAS
FILTER status IN (POWER, ACTIVE, LESS_ACTIVE)
PREFERED_ZONE (DRIVER_PREFERENCE(priority=2) UNION HOME_ADDRESS(priority=2) UNION BOOKING_HISTORY(priority=2)
SCORING
(OTD(weight=0.5, threshold>= 0.992, max_value=1.0)
+ Customer_Rating(weight=0.05, threshold>= 0.50, max_value=1.0)
+ Completion_Rate(weight=0.45, threshold>= 0.95, max_value=1.0))
TOP 100
```

- `FILTER status IN (POWER, ACTIVE, LESS_ACTIVE)`: get driver status from table `driver_tags.tag`.
- `PREFERED_ZONE (DRIVER_PREFERENCE(priority=2) UNION HOME_ADDRESS(priority=2) UNION BOOKING_HISTORY(priority=2)`: get driver prefer zone data from collection `driver_prefer_zone`.
- `SCORING (OTD(weight=0.5, ...) + Customer_Rating(weight=0.05, ...) + Completion_Rate(weight=0.45, ...))`: get on-time data; can check in the driver detail screen for each driver.
