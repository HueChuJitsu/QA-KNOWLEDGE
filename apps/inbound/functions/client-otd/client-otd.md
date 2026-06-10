# How New OTD is Calculated

Related tickets: [INB-600](https://gojitsu.atlassian.net/browse/INB-600), [INB-849](https://gojitsu.atlassian.net/browse/INB-849), [INB-860](https://gojitsu.atlassian.net/browse/INB-860)

---

## 1. Old vs New Logic

| | Old Logic | New Logic |
| --- | --- | --- |
| OTD formula | `sort_date + 1 day` | `sort_date + SLA_days` |
| SLA source | Hardcoded (SAME_DAY / NEXT_DAY) | `service_level_agreement_contract` table |
| Flexibility | Fixed for all clients | Per-client contract |

---

## 2. OTD Calculation Formula

```
OTD date = sort_date + SLA_days
```

Where:

- `sort_date` — derived from scan time + sort window (from `sprinkling_config`)
- `SLA_days` — contracted delivery days per client (from `service_level_agreement_contract`)

---

## 3. Step 1 — Determine Sort Date

### Sort window

Each warehouse has a sort window configured in the **`sprinkling_config`** table (e.g. `10:00 PM → 3:00 AM`).

### Rule

> The sort date is always the **calendar date when the sort window started**, even if the actual scan happens after midnight.

### Examples

| Scan Time | Sort Window | In Window? | Sort Date |
| --- | --- | --- | --- |
| 11/20 10:00 PM | 10 PM → 3 AM | ✅ | 11/20 |
| 11/21 1:00 AM | 10 PM → 3 AM | ✅ (after midnight, still in window) | 11/20 |
| 11/21 4:00 AM | 10 PM → 3 AM | ❌ | 11/21 |

### Logic

```
if scan_time is within [window_start, window_end]:
    sort_date = calendar date of window_start
else:
    sort_date = calendar date of scan_time
```

---

## 4. Step 2 — Determine SLA Days

SLA days per client are stored in the **`service_level_agreement_contract`** table.

The system queries this table using the shipment's `clientId` (and optionally `regionCode`) to get the contracted number of delivery days.

---

## 5. Step 3 — Calculate OTD Date

```
OTD date = sort_date + SLA_days
```

### Full example

| Field | Value |
| --- | --- |
| Scan time | 11/21/2025 1:00 AM |
| Sort window | 10:00 PM → 3:00 AM |
| Sort date | 11/20/2025 (window start) |
| SLA days (from contract) | 2 |
| **OTD date** | **11/22/2025** |

---

## 6. Feature Flag

The new OTD logic is gated by a flag — it does **not** replace the old logic immediately for all clients.

| Flag | Type | Where |
| --- | --- | --- |
| `otd_by_sla` | Boolean | Consul — global, all clients |
| `OTD_BY_SLA` | JSON string | MongoDB `client_settings.delivery_settings` — per client |

Regardless of flag state, OTD results are **always written** to `otd_shipment_history` for data analysis. The shipment table is **only updated** when the flag is enabled for that client/region.

For full configuration details, see [INB-849-otd-by-sla-configuration-guide.md](./INB-849-otd-by-sla-configuration-guide.md).

---

## 7. Related References

- [INB-600](https://gojitsu.atlassian.net/browse/INB-600) — OTD calculation SLA table implementation
- [INB-849](https://gojitsu.atlassian.net/browse/INB-849) — Feature flag for gradual rollout
- [INB-860](https://gojitsu.atlassian.net/browse/INB-860) — Sort date determination when scanning shipments
