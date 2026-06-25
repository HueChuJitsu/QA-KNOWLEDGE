# Guessed Shipments Logic

> **Sources:**
> - [Setting related to Guessed Shipments](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1093107743) *(last updated: 2025-01-27)*
> - [Update the number of guessed shipments by client](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1172766767) *(last updated: 2025-04-30)*

---

## 1. Prerequisites

All of the following conditions must be met before the system generates any guessed shipments:

| # | Condition |
|---|-----------|
| 1 | Routing problem must be created **by date** (not by manifest) |
| 2 | Client must be **Commingle** in the `client_service` table |
| 3 | Client must be listed in `include_client` of the `guess_small_parcel` config in `item_metadata` (`owner = RG_{region_code}`, `scope = APP_FEATURES`) |
| 4 | If `day_of_week_enabled` is configured, the selected date's day of week must be in that list — if not, **no guessed shipments are generated** |

### item_metadata config structure

```json
"guess_small_parcel": {
  "type": "java.util.Map",
  "value": "{\"number_of_week\":\"3\", \"include_client\":\"472,655,684\", \"additional_rate\":\"0.5\", \"day_of_week_enabled\": \"1,4,5\", \"guessing_type\":\"client_manifest\"}"
}
```

> `day_of_week_enabled`: `1` = Monday, `7` = Sunday

---

## 2. Priority Chain

Once prerequisites are satisfied, the system follows this priority order to determine the number of guessed shipments:

```
[1] UI Manual Config  ([Configuration] button before [Create])
         ↓  if not configured
[2] client_manifest  (guessing_type = "client_manifest" + guessed_number_shipment table)
         ↓  if not configured or client has no record
[3] number_of_days_before  (overrides number_of_week)
         ↓  if not configured
[4] Date range defaults  (default_days_before / default_days_after)
         ↓  if not configured
[5] number_of_week + additional_rate  (base logic, uses scanned shipments)
```

---

## 3. Priority Details

### [1] UI Manual Config *(highest priority)*

- Router clicks **[Configuration]** before **[Create]** and inputs a shipment count per client
- **Overrides all other logic**
- The number entered is the **exact count** — nothing added on top
- If multiple clients share one total number — system randomly distributes among them, sum = configured number
- Only the configured clients get guessed shipments; all other clients in `include_client` are skipped
- **Cap:** max = total valid shipments in date range (even if the configured number is higher)

---

### [2] client_manifest

**Trigger:** `guessing_type: "client_manifest"` is set in `item_metadata`

- System looks up the `guessed_number_shipment` MongoDB collection by `client_id` + `region` + `delivery_date`
- If a record exists — use `number_shipment` from that record
- If no record exists for a client — fall through to [3] / [4] / [5]

**Collection document structure:**
```json
{
  "number_shipment": 888,
  "client_id": 684,
  "region": "DFW",
  "delivery_date": "2024-08-23T00:00:00.000Z",
  "type": "client_manifest"
}
```

**Example:**
```
Selected date = 2024-08-23 | Region = DFW | No UI config
→ client 684: uses number_shipment = 888 from table
→ client 655: no record → falls back to number_of_week logic
```

---

### [3] number_of_days_before *(overrides `number_of_week`)*

- Clones guessed shipments from shipments with **Valid status** in the past **N days** (including today)
- Valid status excludes: `INVALID_ADDRESS`, `GEOCODE_FAILED`, `UNSERVICEABLE`, `UNDELIVERABLE`
- **Does NOT require** shipments to be scanned in a sort session
- Guess count = total valid shipments ÷ number of days that actually have valid shipment data
- **Cap:** max = total valid shipments in the N-day window

---

### [4] Date range defaults

**Trigger:** `default_days_before` and/or `default_days_after` set in `item_metadata`

- System first checks if API passes `time_range_start` / `time_range_end` params — uses those
- If not, falls back to `default_days_before` + `default_days_after` from `item_metadata`
- Fetches valid shipments in the range (including current date), calculates average + `additional_rate`
- **Cap:** max = total valid shipments in the date range

**Example config:**
```json
{
  "number_of_days_before": "1",
  "number_of_week": "2",
  "default_days_before": "2",
  "default_days_after": "2",
  "include_client": "472,655,519",
  "additional_rate": "0.5",
  "day_of_week_enabled": "1,2,3,4,5,6,7"
}
```

---

### [5] number_of_week + additional_rate *(base logic / last fallback)*

- Uses **scanned shipments in sort sessions** with the same day-of-week as the routing date, over the past N weeks
- X = average scanned shipments across those N weeks
- Guess count = `floor(X) + floor(X × additional_rate)`
- **Requires** the client to have scanned shipments in sort session
- **Cap:** max = total scanned shipments in the reference period

---

## 4. Universal Cap Rule

> Regardless of which tier is used, **the number of generated guesses never exceeds the total valid/scanned shipments in the corresponding date range** — even when a higher number is manually configured via UI.

---

## 5. Guessed Shipments Sprinkle *(optional config)*

To allow one-off sprinkle of a real shipment into a route that already has guessed shipments, configure at least one of the following in `item_metadata`:

```json
"guessed_route_additional_shipment": {
  "type": "java.lang.Integer",
  "value": "2"
},
"guessed_route_additional_volume": {
  "type": "java.lang.Double",
  "value": "2"
}
```

When set, the system **ignores the capacity** of guessed shipments during the one-off sprinkle of real shipments.

---

## 6. Quick Reference: Config Fields

| Field | Description | Overrides |
|-------|-------------|-----------|
| `include_client` | Comma-separated list of eligible client IDs | — |
| `day_of_week_enabled` | Days guessing is active (1=Mon, 7=Sun) | — |
| `number_of_guess` *(UI)* | Manual override per client, exact count | All logic |
| `guessing_type: client_manifest` | Enables lookup from `guessed_number_shipment` table | `number_of_week` for matched clients |
| `number_of_days_before` | Look back N days, valid status, no sort scan needed | `number_of_week` |
| `default_days_before` / `default_days_after` | Date range around today for valid shipment lookup | `number_of_week` |
| `number_of_week` | Look back N same-day-of-week, requires sort scan | — |
| `additional_rate` | Multiplier added on top of base average | — |
| `guessed_route_additional_shipment` | Allow sprinkle ignoring shipment capacity | — |
| `guessed_route_additional_volume` | Allow sprinkle ignoring volume capacity | — |
