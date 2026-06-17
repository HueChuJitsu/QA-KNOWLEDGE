# Geofence — Enable / Disable Guide

How to enable and disable geofencing on Jitsu.

---

## Scope

Geofencing can be enabled/disabled at three levels:

- **Ticket** (`TK_{ticket-id}`)
- **Assignment**
- **Stop** (shipment)

Assignment and Stop can be toggled directly on the **Dispatch UI**.
Ticket level has **no UI** — it must be done via API.

---

## 1. Enable / Disable on UI

> Available for **Assignment** and **Stop** only.

### Assignment

1. Locate the assignment → open **Assignment detail**.
2. Click the **geofence** button.

### Stop

1. Locate the assignment → open **Assignment detail**.
2. Click any shipment to open **Shipment detail** (stop detail).
3. Click the **geofence** button.

> The geofence button is in the toolbar at the bottom of the Assignment / Shipment detail panel (📍 pin icon).

---

## 2. Enable / Disable for a Ticket (API only)

There is no UI to enable/disable geofencing at the ticket level — use the metadata API.

**Name:** update metadata

```
PUT {{driver-app-api-url}}/metadata/TK_{ticket-id}/DELIVERY/geofencing
```

**Body:**

```json
{
  "type": "String",
  "value": "true/false"
}
```

- `value = "true"` → enable geofencing
- `value = "false"` → disable geofencing

Replace `{ticket-id}` with the target ticket id (path becomes `TK_{ticket-id}`).
