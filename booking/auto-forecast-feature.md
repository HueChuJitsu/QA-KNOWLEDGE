# Auto forecast feature

> Source: [Confluence — Auto forecast feature](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2218786862/Auto+forecast+feature)

## Auto Forecast (Booking) — User Manual

### 1) Purpose

Auto Forecast automatically calculates **forecast ticket numbers** and **tour cost ranges (min/max)** per **Zone** using historical booking sessions. Forecast outputs are written to a dedicated V2 collection and can be reset back to system-calculated values after manual edits.

### 2) Enable Auto Forecast (per region)

Auto Forecast is enabled via **item_metadata**.

**Required metadata:**

- `owner`: `RG_{region}`
- `scope`: `BOOKING`
- Setting:

```json
"use_booking_forecast_v2": {
  "type": "java.lang.String",
  "value": "true"
}
```

### 3) Data storage (Collections)

**V2 forecast output**

- `booking_forecast_v2` — primary collection that stores forecast results per zone.

**Historical + intermediate data**

- `booking_summary` — stores historical booking session data used for forecast calculations.
- `booking_forecast_data` — stores calculated forecast data snapshots **after a successful update** to `booking_forecast_v2`.
- `booking_forecast` — existing collection that stores calculated data (kept as part of the current pipeline).

**Holiday / no-service dates**

- `no_service_date` — defines dates with no service.

### 4.1 Worker workflow (Updated — worker name clarified)

**Worker name:** **Create Advance Booking Session**

#### Trigger schedule (attributes)

When **Create Advance Booking Session** is triggered by schedule, it can include additional attributes:

```json
"booking_summary": "true",
"backfilling_days": "x"
```

**Meaning**

- `booking_summary = "true"`: selected historical data is saved into `booking_summary`, calculated data is saved in `booking_forecast_data`, and the respective `booking_forecast_v2` is updated. Otherwise, the Worker just creates the booking session normally as in the old logic.
- `backfilling_days = "x"`: number of days to look back when selecting historical booking sessions.

#### Worker flow

1. **Load the created** `booking_session` and extract:
   - **Regions**
   - **Ticket type**
2. **Query historical booking sessions** that satisfy **all** conditions:
   - `status` ∈ { `In_progress`, `Completed` }
   - `region` is in **Regions** from Step 1
   - `ticket_type` equals the **Ticket type** from Step 1
   - `target_date` is within the backfilling window:
     - Window end = **worker_run_date − 1 day**
     - Window start = (worker_run_date − 1 day) − `backfilling_days`
3. If `booking_summary = "true"`, **write selected sessions** into `booking_summary` (still saved even if zone data is 0, as long as there is a `ticket_book.attributes.zone_id` with a correct value).
4. **Calculate forecast-related data** from `booking_summary` and **update** `booking_forecast_v2`.
5. **If Step 4 succeeds**, save the calculated snapshot into `booking_forecast_data`.

### 5) Forecast calculations in collection `booking_forecast_data`

#### 5.1 Rolling X-week ticket number (per zone)

**Definition:** rolling X-week ticket number for a zone is the **average weekly routes** across the last **X weeks** (X from [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/booking_service/booking_forecast_number_of_weeks/edit)).

**Important rule**

- The average divides **only by the number of weeks that actually have routes for that zone**.
- If the calculated result = 0, this field is not saved.

**Formula**

- `rolling_x_week_tickets(zone) = total routes in X weeks / count(weeks_with_tickets_for_zone)`

#### 5.2 Average ticket (per zone)

**Definition**

- `average_ticket = last_week_total_routes * percentage_in_consul`

_Where:_

- `last_week_total_routes` = number of tickets of a zone in the **latest week that has that zone's data (greater than 0), within the X-week window** set in [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/booking_service/booking_forecast_number_of_weeks/edit), derived from `booking_summary`.
- `percentage_in_consul` = percentage setting from [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/booking_service/booking_forecast_percentage/edit).

_Note:_

- This field is not saved if the calculated result = 0.
- If `average_ticket = 0` → the field `average_ticket` in `booking_forecast_v2` is not updated.

#### 5.3 Last total tickets (per zone)

- `last_total_tickets = latest_week_total_routes`

_Note:_ always present, as it is just read from collection `booking_summary`.

#### 5.4 Min/Max tour cost (per zone) — based on historical sessions

Auto Forecast calculates **min_tour_cost** and **max_tour_cost** for each zone using **historical booking session data** in `booking_summary`, within the last **X weeks**.

For each zone within the rolling **X-week** window:

- `min_tour_cost` = **minimum tour cost observed**
- `max_tour_cost` = **maximum tour cost observed**

### 6) User action: Reset ticket number (per zone)

If a user manually edits (whether clicking the **[Save]** button or not) the ticket number for a zone and then chooses **Reset**:

- The system resets that zone's ticket number back to the **calculated** `average_ticket`:
  - `average_ticket = latest_week_total_routes * percentage_in_consul`

### 7) Holiday / No-service date setting

Holiday logic is maintained via collection `no_service_date`.

**Priority**

1. **Region-level** configuration.
2. **All-region** configuration (`region = DEFAULT`).

If a region-specific entry exists for a date, it overrides the all-region default for that date.

_**Note:**_ Independence day is July 4th on the calendar, but our business is actually off on July 5th, so the Booking session is actually not created on July 5th.
