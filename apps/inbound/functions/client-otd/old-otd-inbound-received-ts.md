# OTD Calculation — Old Logic (inbound_received_ts)

> **App:** Inbound · **Worker:** `ScannedShipmentEventHandler`
> **Status:** Superseded by the new logic (INB-600)
> **New logic:** see [otd-by-sla.md](./otd-by-sla.md)

---

## 1. Overview

The old logic computes OTD based on **`inbound_received_ts`** — the time the shipment is received at the warehouse — without going through the SLA contract.

| | Old logic | New logic (INB-600) |
|---|---|---|
| Time source | `inbound_received_ts` | `inbound_first_scan_ts` / `container_session` / `inferred_arrival_ts` |
| SLA days | Hardcoded = **1** | From the SLA API per client |
| Branching | Late/OnTime vs Early | No branching |
| Blame check | ✅ Yes | ❌ No |
| maxOtdResetDays | ✅ Yes (per-client) | ❌ No |
| Worker | `ScannedShipmentEventHandler` | `OTDCalculatedHandler` |

---

## 2. Trigger Conditions

The logic runs every time a shipment is scanned in the warehouse (event `INBOUND.SCAN`), except:

- Scan type = **PRESORT** → runs the `updateDateWhenPresort` logic instead
- `shipment.timezone = null` → **SKIP**
- `shipment.inbound_received_ts = null` → **SKIP**

---

## 3. General Formula

```
OTD date = calculateOtdDate(inbound_received_ts) + 1 day
```

The constant `DAY_PLUS = 1` is hardcoded and applies to all clients / service levels.

---

## 4. Sort Date Function — `calculateOtdDate()`

Pushes any timestamp through the sort window to produce a sort date:

```
scannedDateZ = scannedTime.startOfDay - 1 day
# Looked up by clientId in sprinkling_config.clients; if the client is not
# configured there, fall back to the defaults below.
start = sprinkling_config.clients[clientId].sort_start  (default: PT14H)
end   = sprinkling_config.clients[clientId].sort_end    (default: PT29H)

days = 0
while (end + days days) < scannedTime:
    days += 1

sort_date = scannedDateZ + days
```

### Config source

The sort window is looked up **per client** in `sprinkling_config.clients`. If the shipment's `clientId` is **not configured** in `sprinkling_config.clients`, OTD is **always** computed with the default window `PT14H – PT29H` (14:00 → 05:00 next day).

| Condition | sort_start | sort_end |
|---|---|---|
| `clientId` not set in `sprinkling_config.clients` | **PT14H (14:00)** — default | **PT29H (05:00 next day)** — default |
| Client configured but key missing | Fallback PT14H | Fallback PT29H |
| Client fully configured | `attributes.sort_start` | `attributes.sort_end` |

### Examples

| Scan time | sort_end | sort_date |
|---|---|---|
| 11/21 1:00 AM (25H) | PT29H (5:00 AM) | 11/20 (still within window) |
| 11/21 6:00 AM (30H) | PT29H (5:00 AM) | 11/21 (outside window → new day) |

---

## 5. Main Branching Logic — `updateOtd()`

### Step 1 — Compute reference points

```
inboundReceivedTsZ             = inbound_received_ts (in the shipment's timezone)
inboundReceivedCalculatedZ     = calculateOtdDate(inboundReceivedTsZ)
inboundReceivedCalculatedSnapZ = inboundReceivedCalculatedZ.startOfDay

originalDropoffEarliestTsSnapZ = originalDropoffEarliestTs.startOfDay
```

### Step 2 — Compare to branch

```
if inboundReceivedCalculatedSnapZ >= originalDropoffEarliestTsSnapZ - 1 day:
    → Case 1: Received LATE or ON TIME

else:
    → Case 2: Received EARLY
```

---

### Case 1: Received LATE or ON TIME

The shipment arrives late or on time relative to the original dropoff window.

```
1. Blame check:
   if isAxlehireBlame(billing_dispositions) == true:
       → SKIP, do not update OTD (AxleHire's fault, do not push OTD out)

2. Compute the new OTD:
   estimatedOtdDateZ = inboundReceivedCalculatedZ + 1 day

   otd_earliest_ts = estimatedOtdDateZ
   otd_latest_ts   = estimatedOtdDateZ
```

**`isAxlehireBlame`:** takes the latest `BillingDisposition` with status `APPROVED` and checks whether `inbound_blame = "AXLEHIRE"`.

---

### Case 2: Received EARLY

The shipment arrives earlier than expected (before `originalDropoffEarliestTs - 1 day`).

```
estimatedOtdDateZ = calculateOtdDate(inboundScanTs) + 1 day

maxOtdResetDays   = client_settings.delivery_settings.max_otd_reset_days
                    (default = 2 if not configured)

maxOtdDateZ = inboundReceivedCalculatedZ + maxOtdResetDays

otd_earliest_ts = min(estimatedOtdDateZ, min(originalDropoffEarliestTs, maxOtdDateZ))
otd_latest_ts   = otd_earliest_ts
```

**Meaning of `maxOtdResetDays`:** caps how many days the OTD can be reset relative to `inbound_received_ts` — preventing the OTD from being pushed too far out when a shipment arrives very early.

---

## 6. Final Step — `resetOtdTime()`

After the OTD date is computed, the specific time-of-day is re-applied based on the `ClientService`:

| Service type | Earliest time | Latest time |
|---|---|---|
| COMMINGLE + NEXT_DAY | `clientService.dropoffEarliest` (default 8:00) | `clientService.dropoffLatest` (default 20:00) |
| Other cases | Keep hh:mm:ss from `originalDropoffEarliestTs` | Keep hh:mm:ss from `originalDropoffLatestTs` |

If no `ClientService` is found → use `originalDropoffEarliestTs` / `originalDropoffLatestTs` as-is.

---

## 7. Full Examples

### Case 1: Received Late

| Field | Value |
|---|---|
| `inbound_received_ts` | 11/22 2:00 AM |
| `sort_end` | PT29H (5:00 AM) |
| `inboundReceivedCalculatedZ` | 11/21 (within sort window 10PM–5AM) |
| `originalDropoffEarliestTs` | 11/21 8:00 AM |
| Comparison | 11/21 >= 11/21 - 1 day → **LATE** |
| `isAxlehireBlame` | false |
| `estimatedOtdDateZ` | 11/21 + 1 = **11/22** |
| **Result** | `otd_earliest_ts = otd_latest_ts = 11/22 8:00 AM` |

### Case 2: Received Early

| Field | Value |
|---|---|
| `inbound_received_ts` | 11/18 2:00 AM |
| `inboundReceivedCalculatedZ` | 11/17 |
| `originalDropoffEarliestTs` | 11/22 8:00 AM |
| Comparison | 11/17 < 11/22 - 1 day (11/21) → **EARLY** |
| `inboundScanTs` | 11/18 3:00 AM → `calculateOtdDate` = 11/17 |
| `estimatedOtdDateZ` | 11/17 + 1 = 11/18 |
| `maxOtdResetDays` | 2 |
| `maxOtdDateZ` | 11/17 + 2 = 11/19 |
| `otd_earliest_ts` | min(11/18, min(11/22, 11/19)) = **11/18** |
| **Result** | `otd_earliest_ts = otd_latest_ts = 11/18 8:00 AM` |

---

## 8. Data Locations

| Item | Location |
|---|---|
| Sort window config | `sprinkling_config.clients[clientId].sort_start` / `sort_end` (MongoDB); defaults PT14H / PT29H if client not configured |
| Max OTD reset days | `client_settings.delivery_settings.max_otd_reset_days` (MongoDB) |
| Blame check | `billing_disposition` table (PostgreSQL) |
| ClientService (dropoff times) | `client_service` table (PostgreSQL) |
| Result written to | `shipments.otd_earliest_ts` / `otd_latest_ts` (PostgreSQL) |

---

## 9. Related References

- [INB-600](https://gojitsu.atlassian.net/browse/INB-600) — New logic that supersedes this
- [otd-by-sla.md](./otd-by-sla.md) — New logic (sort_date + SLA_days) + feature-flag rollout config
