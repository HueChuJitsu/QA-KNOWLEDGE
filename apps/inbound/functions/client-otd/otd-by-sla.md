# Client OTD — OTD by SLA (New Logic)

> **App:** Inbound · **Function:** Client OTD (OTD by Client) · **Status:** Production
> **Worker:** `OTDCalculatedHandler` · **Affected service:** `event-processor-shipment`
> **Old logic (deprecated):** see [old-otd-inbound-received-ts.md](./old-otd-inbound-received-ts.md)
> **Tickets:** [INB-600](https://gojitsu.atlassian.net/browse/INB-600), [INB-849](https://gojitsu.atlassian.net/browse/INB-849), [INB-860](https://gojitsu.atlassian.net/browse/INB-860)

---

## 1. Overview

OTD by SLA computes the OTD date as `sort_date + SLA_days`, where `SLA_days` comes from each client's contract (SLA API / `service_level_agreement_contract` table) instead of a hardcoded service level. This fixes **Standard Service** contracts with 2+ day delivery windows, which the old logic could not handle.

Because OTD directly drives the SLA reports sent to clients, the new logic is gated by a flag (INB-849): results are **always written** to `otd_shipment_history` for comparison, but the shipment table is **only updated** when enabled for that client/region.

| | Old logic | New logic (this doc) |
|---|---|---|
| OTD formula | `sort_date + 1 day` | `sort_date + SLA_days` |
| SLA source | Hardcoded (SAME_DAY / NEXT_DAY) | `service_level_agreement_contract` table |
| Flexibility | Fixed for all clients | Per-client contract |

---

## 2. Calculation

```
OTD date = sort_date + SLA_days
```

- `sort_date` — derived from scan time + sort window (`sprinkling_config`).
- `SLA_days` — contracted delivery days per client (`service_level_agreement_contract`), queried by `clientId` (and optionally `regionCode`).

### Step 1 — Determine sort date

Each warehouse has a sort window in `sprinkling_config` (e.g. `10:00 PM → 3:00 AM`).

> The sort date is always the **calendar date when the sort window started**, even if the actual scan happens after midnight.

```
if scan_time is within [window_start, window_end]:
    sort_date = calendar date of window_start
else:
    sort_date = calendar date of scan_time
```

| Scan Time | Sort Window | In Window? | Sort Date |
| --- | --- | --- | --- |
| 11/20 10:00 PM | 10 PM → 3 AM | ✅ | 11/20 |
| 11/21 1:00 AM | 10 PM → 3 AM | ✅ (after midnight, still in window) | 11/20 |
| 11/21 4:00 AM | 10 PM → 3 AM | ❌ | 11/21 |

### Step 2 — Add SLA days

| Field | Value |
| --- | --- |
| Scan time | 11/21/2025 1:00 AM |
| Sort window | 10:00 PM → 3:00 AM |
| Sort date | 11/20/2025 (window start) |
| SLA days (from contract) | 2 |
| **OTD date** | **11/22/2025** |

---

## 3. Configuration (Feature Flag)

### 3.1 Levels & precedence

| Level | Applies to | Storage | Default |
| --- | --- | --- | --- |
| Global | All clients, all regions | Consul | `false` |
| Per-client | One client, by region | MongoDB `client_settings.delivery_settings` | Not set = disabled |

```
Global (Consul) = true  →  enable for ALL clients/regions; per-client config ignored
Global (Consul) = false →  check per-client config for each shipment
Per-client not set      →  disabled for that client
```

### 3.2 `otd_by_sla` — Global flag (Consul)

Boolean, used by `event-processor-shipment`. Default `false`. **Restart required** after change (read at startup).

```text
# Production
prod/apps/workers/event_processors/shipment/attributes/otd_by_sla
# Staging
staging/apps/workers/event_processors/shipment/attributes/otd_by_sla
```

### 3.3 `OTD_BY_SLA` — Per-client (MongoDB `client_settings.delivery_settings`)

JSON string with `enabled_regions` (array) + `all_regions` (boolean). Not set = disabled. **No restart** — cache TTL 5 min.

```json
// All regions
"OTD_BY_SLA": "{\"enabled_regions\": [], \"all_regions\": true}"
// Specific regions
"OTD_BY_SLA": "{\"enabled_regions\": [\"CHI\"], \"all_regions\": false}"
```

### 3.4 Behavior per state

| Consul `otd_by_sla` | `OTD_BY_SLA` | `otd_shipment_history` | Shipment table updated |
| --- | --- | --- | --- |
| `true` | any | ✅ Always written | ✅ All clients / all regions |
| `false` | `all_regions: true` | ✅ Always written | ✅ That client, all regions |
| `false` | `enabled_regions: ["CHI"]` | ✅ Always written | ✅ That client, CHI only |
| `false` | Not set | ✅ Always written | ❌ Not updated |

> `otd_shipment_history` is **always written** for all shipments regardless of config — used by the data team to compare against the old logic.

### 3.5 Apply / restart

| Change | Restart `event-processor-shipment`? | Reason |
| --- | --- | --- |
| Consul `otd_by_sla` | **Yes** | Consul key read at startup |
| `delivery_settings` (`OTD_BY_SLA`) | **No** | Cache TTL 5 min auto-expires |

---

## 4. Database / Schema Changes

| Item | Detail |
| --- | --- |
| New collection | `otd_shipment_history` |
| Renamed from | `otd_assignment_history` (still exists, historical data, not dropped) |
| New fields | `injection_region`, `timezone` |

---

## 5. Validation & Troubleshooting

```javascript
// OTD history is being written
db.otd_shipment_history.find({ shipmentId: <shipment_id> }).sort({ _id: -1 }).limit(1)

// Client setting is applied
db.client_settings.findOne({ clientId: <client_id> }, { "delivery_settings.OTD_BY_SLA": 1 })
```

- `Failed to parse OTD_BY_SLA config for client X` → malformed JSON string, check escape characters.
- `delivery_settings` change not taking effect → wait 5 minutes for the cache to expire.

---

## 6. Related References

- [old-otd-inbound-received-ts.md](./old-otd-inbound-received-ts.md) — old logic (deprecated)
- [INB-600](https://gojitsu.atlassian.net/browse/INB-600) — OTD calculation via SLA table
- [INB-849](https://gojitsu.atlassian.net/browse/INB-849) — Feature flag for gradual rollout
- [INB-860](https://gojitsu.atlassian.net/browse/INB-860) — Sort date determination when scanning shipments
- [OTD Calculation Updates](https://gojitsu.atlassian.net/wiki/spaces/DOJO/pages/1663139850/OTD+Calculation+Updates) — Confluence
- [OTD by SLA — Configuration Guide](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2565734414/OTD+by+SLA+Configuration+Guide) — Confluence
