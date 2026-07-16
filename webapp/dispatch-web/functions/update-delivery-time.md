# Function: Update Delivery Time

> **Read from (on submit):** `client_service` (`dropoff_earliest`, `dropoff_latest`)
> **Read from (after routing):** `client_settings.delivery_settings` (`dropoff_earliest`, `dropoff_latest`)
> **Write to:** `shipments` table (`dropoff_earliest_ts`, `dropoff_latest_ts`)

## 1. Description
<!-- What "Update Delivery Time" does, which actor triggers it, and its purpose. -->

The shipment's delivery-time window (`dropoff_earliest_ts` / `dropoff_latest_ts` on the `shipments` table) is set in **two stages** of its lifecycle:

1. **On submit** — computed from the client's service configuration (`client_service`), with an override for business addresses.
2. **After routing** — (re)computed from `client_settings.delivery_settings`; if not configured there, defaults to **8AM–8PM**.

## 2. Business Flow
<!-- Main steps, entry & exit conditions. -->

**Stage 1 — on submit:**
1. Shipment is created and **submitted**.
2. System looks up the matching `client_service` record and reads `dropoff_earliest` / `dropoff_latest` (business-address override applies — see section 3.2).
3. Updates the shipment's `dropoff_earliest_ts` / `dropoff_latest_ts`.

**Stage 2 — after routing:**
4. After routing completes, the values are taken from `client_settings.delivery_settings`.
5. If `delivery_settings` is not configured, defaults to **8AM–8PM**.

```
shipment submitted
      │  dropoff_earliest_ts / dropoff_latest_ts
      │  ← client_service (or business-address override, §3.2)
      ▼
   routing
      │  dropoff_earliest_ts / dropoff_latest_ts
      │  ← client_settings.delivery_settings (§3.4)
      │     default 8AM–8PM if not configured
      ▼
   shipments (final delivery window)
```


### 3.1 Field mapping (default — non-business address)

Applies when the dropoff address is **not** a business (`is_business = false`). For business addresses, see section 3.2 which **overrides** these values.

| Source (`client_service`) | Target (`shipments`) |
|---------------------------|----------------------|
| `dropoff_earliest` | `dropoff_earliest_ts` |
| `dropoff_latest` | `dropoff_latest_ts` |

### 3.2 Business address override

If the dropoff address is a **business** (`is_business = true`), the delivery window is taken from **business hours** instead of the `client_service` values:

| Field | Value when `is_business = true` |
|-------|---------------------------------|
| `dropoff_earliest_ts` | **8:00 AM** |
| `dropoff_latest_ts` | the address's **latest business time** |

```
is_business ?
  ├── YES → dropoff_earliest_ts = 8AM
  │         dropoff_latest_ts   = latest business time
  └── NO  → dropoff_earliest_ts = client_service.dropoff_earliest
            dropoff_latest_ts   = client_service.dropoff_latest
```

### 3.3 Same-day vs Next-day

Classification is based on the shipment's **delivery date** compared to the **current date** (the date it is submitted):

| Type | Condition |
|------|-----------|
| **Same day** | `delivery_date == current_date` |
| **Next day** | `delivery_date != current_date` — i.e. any date **other than** today, whether in the **past** or the **future** |

> **Note:** "Next day" here does **not** literally mean "tomorrow" — it means the delivery date differs from the current date. A shipment submitted with a past or a future delivery date is treated as **next day**.

### 3.4 After routing — from `client_settings.delivery_settings`

Once routing completes, `dropoff_earliest_ts` / `dropoff_latest_ts` are (re)computed from `client_settings.delivery_settings`, using its `dropoff_earliest` / `dropoff_latest` values.

| Source (`client_settings.delivery_settings`) | Target (`shipments`) |
|----------------------------------------------|----------------------|
| `dropoff_earliest` | `dropoff_earliest_ts` |
| `dropoff_latest` | `dropoff_latest_ts` |

**Default** — if `delivery_settings` is not configured, the window defaults to **8AM–8PM**.

Values are expressed as an **hour-of-day** (e.g. `"8H"` = 8:00 AM, `"22H"` = 10:00 PM):

```json
{
  "dropoff_earliest": "8H",
  "dropoff_latest": "22H"
}
```

```
after routing → client_settings.delivery_settings configured ?
  ├── YES → dropoff_earliest_ts = delivery_settings.dropoff_earliest
  │         dropoff_latest_ts   = delivery_settings.dropoff_latest
  └── NO  → default 8AM–8PM
```

### 3.5 Trigger

- **Stage 1** runs **after the shipment is submitted** (not on create/draft).
- **Stage 2** runs **after routing completes**.


## 4. QA / Test notes
<!-- Happy cases, edge cases, sample data, things to watch when testing. -->

- **Non-business address (default):** submit a shipment to a residential address → `dropoff_earliest_ts` / `dropoff_latest_ts` come from `client_service.dropoff_earliest` / `dropoff_latest`.
- **Business address override:** submit a shipment where the dropoff address is `is_business = true` → `dropoff_earliest_ts = 8AM`, `dropoff_latest_ts = latest business time` (client_service values are **not** used).
- **Business boundary:** address with `is_business = true` but no / unusual business hours → confirm `latest business time` resolution and the 8AM earliest.
- **After routing — configured:** `client_settings.delivery_settings` set (e.g. `dropoff_earliest: "8H"`, `dropoff_latest: "22H"`) → after routing, `dropoff_earliest_ts` / `dropoff_latest_ts` reflect those hours.
- **After routing — default:** `delivery_settings` not configured → window defaults to **8AM–8PM**.
- **Stage precedence:** confirm whether the after-routing value overwrites the on-submit value (client_service / business override).
- **Trigger guard:** Stage 1 runs on submit, Stage 2 after routing — confirm a not-yet-submitted / not-yet-routed shipment is not updated in the respective stage.
- **Missing config:** non-business address with no matching `client_service`, or `dropoff_earliest` / `dropoff_latest` null → confirm expected behavior (skip / default / error).
- **Downstream:** confirm the new `dropoff_earliest_ts` / `dropoff_latest_ts` propagate to consumers (e.g. OTD `min_dropoff_earliest_ts` / `deliveryByTs`).

