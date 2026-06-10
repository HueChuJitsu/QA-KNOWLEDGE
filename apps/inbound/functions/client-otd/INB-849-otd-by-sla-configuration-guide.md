# OTD by SLA — Configuration Guide

> **App:** Inbound · **Function:** Client OTD (OTD by Client) · **Status:** Production
> **Shared context:** see [../../README.md](../../README.md) · **Calculation logic:** see [client-otd.md](./client-otd.md)

Configuration guide for enabling/disabling the new OTD calculation logic (based on SLA table) per client and region, without a code deploy. Related ticket: [INB-849](https://gojitsu.atlassian.net/browse/INB-849).

---

## 0. Background

The old OTD logic calculated based on **service level** (SAME_DAY / NEXT_DAY), hardcoding the assumption that all shipments deliver within one or two days. This broke down for **Standard Service** contracts with 2+ day delivery windows.

The new logic ([INB-600](https://gojitsu.atlassian.net/browse/INB-600)) replaces this by querying the **SLA API** to get the exact delivery window per client contract — consistent across all service types.

However, since OTD directly affects SLA reports sent to clients, it cannot be enabled immediately for everyone. INB-849 introduces a flag mechanism:

- **Always calculate and save** new OTD results to `otd_shipment_history` → data team can compare against old logic
- **Only update the shipment table** when explicitly enabled via config → no client impact until verified

---

## 1. Overview

| Attribute | Value |
| --- | --- |
| Feature / Config | OTD by SLA |
| Affected services | `event-processor-shipment` |
| Related tickets | [INB-849](https://gojitsu.atlassian.net/browse/INB-849), [INB-600](https://gojitsu.atlassian.net/browse/INB-600) |
| Status | Production |

---

## 2. Configuration Levels & Precedence

| Level | Applies to | Storage | Default |
| --- | --- | --- | --- |
| Global | All clients, all regions | Consul | `false` |
| Per-client | One specific client, by region | MongoDB `client_settings.delivery_settings` | Not set = disabled |

**Resolution order:**

```
Global (Consul) = true  →  enable for ALL clients and regions, per-client config is ignored
Global (Consul) = false →  check per-client config for each shipment
Per-client not set      →  disabled for that client
```

---

## 3. Configuration Reference

### 3.1 `otd_by_sla` — Global flag

| Field | Value |
| --- | --- |
| Type | Boolean |
| Storage | Consul |
| Used by | `event-processor-shipment` |
| Purpose | Enable OTD by SLA for all clients and regions |
| Default | `false` |
| Allowed values | `true` / `false` |
| Configurable at | Global only |
| Required before | Restart `event-processor-shipment` after changing |

Consul path:

```text
# Production
prod/apps/workers/event_processors/shipment/attributes/otd_by_sla

# Staging
staging/apps/workers/event_processors/shipment/attributes/otd_by_sla
```

---

### 3.2 `OTD_BY_SLA` — Per-client config

| Field | Value |
| --- | --- |
| Type | JSON string |
| Storage | MongoDB `client_settings.delivery_settings` |
| Used by | `event-processor-shipment` |
| Purpose | Enable OTD by SLA for a specific client, by region or all regions |
| Default | Not set = disabled for that client |
| Allowed values | JSON with `enabled_regions` (array) and `all_regions` (boolean) |
| Configurable at | Per-client |
| Cache TTL | 5 minutes — no restart needed, wait 5 min after updating |

Enable for all regions:

```json
"OTD_BY_SLA": "{\"enabled_regions\": [], \"all_regions\": true}"
```

Enable for specific regions:

```json
"OTD_BY_SLA": "{\"enabled_regions\": [\"CHI\"], \"all_regions\": false}"
```

MongoDB structure:

```json
{
  "delivery_settings": {
    "OTD_BY_SLA": "{\"enabled_regions\": [\"CHI\"], \"all_regions\": false}"
  }
}
```

---

## 4. Behavior per State

| Consul `otd_by_sla` | `OTD_BY_SLA` in delivery_settings | `otd_shipment_history` | Shipment table updated |
| --- | --- | --- | --- |
| `true` | any | ✅ Always written | ✅ All clients / all regions |
| `false` | `all_regions: true` | ✅ Always written | ✅ That client, all regions |
| `false` | `enabled_regions: ["CHI"]` | ✅ Always written | ✅ That client, CHI only |
| `false` | Not set | ✅ Always written | ❌ Not updated |

> `otd_shipment_history` is **always written** for all shipments regardless of config — used by the data team for analysis and comparison with old logic.

---

## 5. Database / Schema Changes

| Item | Detail |
| --- | --- |
| New collection | `otd_shipment_history` |
| Renamed from | `otd_assignment_history` (still exists, contains historical data, not dropped) |
| New fields | `injection_region`, `timezone` |

---

## 6. Apply / Restart Checklist

| Service | Type | Restart required? | Reason | Order |
| --- | --- | --- | --- | --- |
| `event-processor-shipment` | Worker | **Yes** — when changing Consul flag | Consul key read at startup | 1 |
| `event-processor-shipment` | Worker | **No** — when changing `delivery_settings` | Cache TTL 5 min auto-expires | — |

---

## 7. Validation & Troubleshooting

Verify OTD history is being written:

```javascript
db.otd_shipment_history.find({ shipmentId: <shipment_id> }).sort({ _id: -1 }).limit(1)
```

Verify client setting is applied:

```javascript
db.client_settings.findOne(
  { clientId: <client_id> },
  { "delivery_settings.OTD_BY_SLA": 1 }
)
```

Common errors:

- `Failed to parse OTD_BY_SLA config for client X` → JSON string is malformed, check escape characters
- `delivery_settings` change not taking effect immediately → wait 5 minutes for cache to expire

---

## 8. Related References

- [INB-849](https://gojitsu.atlassian.net/browse/INB-849) — Feature ticket
- [INB-600](https://gojitsu.atlassian.net/browse/INB-600) — OTD calculation SLA table implementation
- [OTD Calculation Updates](https://gojitsu.atlassian.net/wiki/spaces/DOJO/pages/1663139850/OTD+Calculation+Updates) — Confluence
- [OTD by SLA — Configuration Guide](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2565734414/OTD+by+SLA+Configuration+Guide) — Confluence (this doc)
