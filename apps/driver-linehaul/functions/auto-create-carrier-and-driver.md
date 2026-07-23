# Auto-Create Carrier and Driver

## 1. Overview

A feature that automatically scans NetSuite daily and creates LINEHAUL driver accounts without any manual intervention.
Drivers assigned to upcoming loads will have an account ready to log in to the Linehaul Driver App before the pickup date.

**Confluence doc:** [Linehaul Driver Auto-Credentialing — Configuration Guide](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/2578645008)

---

## 2. Feature: Auto-Create Carrier (Courier)

### 2.1 Purpose

Before creating a driver, the system needs a **Courier record** corresponding to the carrier from NetSuite.
The worker automatically creates a Courier if it does not already exist in the DB — this is a prerequisite for `ThirdPartyDriverManager` credentialing to succeed.

### 2.2 Data source from NetSuite

Carrier data is taken from `motorFreightCarrierDetails` → `TmsCarrier` in the NetSuite payload:

| NetSuite field | Courier field | Description |
| --- | --- | --- |
| `TmsCarrier.name` | `company` | Carrier name |
| `TmsCarrier.carrierDOT` | — | DOT number (stored in `linehaul.carrierDot`) |
| `TmsCarrier.carrierMC` | — | MC number (stored in `linehaul.carrierMc`) |
| `MotorFreightCarrierDetails.driverName` | — | Driver name |
| `MotorFreightCarrierDetails.driverPhone` | — | Driver phone (used to create the account) |

### 2.3 Courier creation logic

```
for each linehaul in window:
  if driverPhone is blank → SKIP (no carrier, no driver, no Slack)
  
  lookup Courier by region + platform = "LINEHAUL"
  if NOT found:
    create new Courier {
      company  = carrierName (from TmsCarrier.name)
      type     = LINEHAUL
      platform = LINEHAUL
      status   = ACTIVE
    }
  
  → proceed to driver credentialing
```

**Important**: If `driverPhone` is blank → the entire flow is skipped before `LinehaulDriverManager` is called. No carrier is created, no driver is created, and no Slack message is sent.

---

## 3. Feature: Driver Auto-Credentialing

### 3.1 High-level flow

```
LinehaulCheckingWorker.process()
  ↓
[raw_linehaul_json present?]
  YES → parse JSON array directly (bypass NetSuite — used for testing)
  NO  → fetchLinehauls() → call NetSuite API
  ↓
provisionLinehaulDrivers()
  ↓ filter: pickupAppointmentDate in [now, now + window_days]
  ↓ dedup by driverPhone (first occurrence wins)
  ↓
for each driver:
  if driverPhone blank → skip silently
  auto-create Courier (if not present)
  ThirdPartyDriverManager.process(ThirdPartyDriverRequest)
    ↓
    duplicate check (phone exists?) → skip + log Slack info
    create User + Driver record
      backgroundStatus = MANUALLY_APPROVED
      status           = BACKGROUND_APPROVED
      registration phases = all FINISHED
    send welcome SMS (forgot password link)
    log success → Slack info channel
    on error    → Slack exception channel
```

### 3.2 Driver is created with these states

| Field | Value | Description |
| --- | --- | --- |
| `backgroundStatus` | `MANUALLY_APPROVED` | Bypass background check |
| `status` | `BACKGROUND_APPROVED` | Ready to operate |
| Registration phases | `FINISHED` | All phases marked complete |
| DL / Birthday | dummy placeholder | Auto-filled, no real value required |

### 3.3 Deduplication

- Within a single run: if the same `driverPhone` appears in multiple loads → only the first occurrence is processed, the rest are skipped
- Across runs: if the phone already exists in the DB → skip + send Slack info

### 3.4 Test data (bypass NetSuite)

Send a message to the `linehaul_checker` queue with `raw_linehaul_json` to bypass NetSuite:

```json
{
  "attributes": {
    "raw_linehaul_json": "[{\"shipmentId\":\"12345\",\"netsuiteRecordData\":{\"linehaulType\":\"FTL\",\"jitsuPO\":\"PO-001\",\"p44Id\":\"\",\"p44URL\":\"\",\"p44PublicURL\":\"\",\"shipType\":\"LH\",\"shipStatus\":\"10 Active\",\"jitsuCustomer\":\"86 Thrive Market\",\"trackOnTime\":\"\",\"totalPkgs\":100,\"totalWeight\":\"5000\",\"mode\":\"TRUCK\",\"trailerId\":\"TR-001\",\"seal\":\"\",\"nsLocation\":\"SFO\",\"dispatchNotes\":\"\"},\"motorFreightCarrierDetails\":{\"equipment\":\"53FT\",\"cost\":\"$1500.00\",\"fuel\":\"$200.00\",\"driverName\":\"John Doe\",\"driverPhone\":\"+14155551234\",\"carrierComments\":\"\",\"carrierLoadId\":\"CL-001\",\"carrierNotesInternal\":\"\",\"detention\":\"\",\"detentionComment\":\"\",\"eldDeviceId\":\"\",\"reeferTemp\":\"\",\"tmsCarrier\":{\"name\":\"ABC Trucking\",\"carrierDOT\":\"DOT123456\",\"carrierMC\":\"MC789012\"}},\"origin\":{\"shipCity\":\"Los Angeles\",\"shipState\":\"CA\",\"shipZip\":\"90001\",\"shipper\":\"Warehouse A\",\"shipAddr1\":\"123 Main St\",\"shipAddr2\":\"\",\"shipContact\":\"Contact A\",\"shipContactPhone\":\"+12135550001\",\"tracking\":{\"pickupAppointmentDate\":\"6/20/2026\",\"pickupAppointmentTime\":\"7:00 AM\",\"pickupAppointmentTimeZone\":\"America/Los_Angeles\",\"pickupAppointmentReference\":\"REF-001\"}},\"destination\":{\"consigneeCity\":\"San Francisco\",\"consigneeState\":\"CA\",\"consigneeZip\":\"94102\",\"consignee\":\"AXL Warehouse B\",\"consigneeAddr1\":\"456 Market St\",\"consigneeAddr2\":\"\",\"consigneeContact\":\"Contact B\",\"consigneeContactPhone\":\"+14155550002\",\"tracking\":{\"deliveryAppointmentDate\":\"6/21/2026\",\"deliveryAppointmentTime\":\"3:00 PM\",\"deliveryAppointmentTimeZone\":\"America/Los_Angeles\",\"deliveryAppointmentReference\":\"REF-002\"}},\"stops\":[]}]"
  },
  "type": "linehaul_checker",
  "scheduleId": "test-001",
  "warehouseId": 1
}
```

---

## 4. Configuration Reference

> **Note**: All config is read in `setup()` at startup → you **must restart** `worker-linehaul-checker` after every change.

### 4.1 `apps.workers.linehaul_checker.driver.linehaul_driver_provisioning_enabled` — New key

| Field | Value |
| --- | --- |
| Type | Boolean |
| Storage | Consul |
| Used by | `worker-linehaul-checker` → `LinehaulCheckingWorker` |
| Purpose | Master gate — turns the entire driver provisioning on/off. When `false`, the worker still runs and processes linehaul data normally but does not create driver accounts. |
| Default | `false` |
| Allowed values | `true` / `false` |
| Configurable at | Global |

**Behavior:**

| Value | Behavior |
| --- | --- |
| `true` | Activates auto-credentialing: scan NetSuite, create courier/driver, send welcome SMS |
| `false` | Disables provisioning; the worker still fetches NetSuite and processes linehaul data as before |

```text
false
```

### 4.2 `apps.workers.linehaul_checker.driver.linehaul_driver_provisioning_window_days` — New key

| Field | Value |
| --- | --- |
| Type | Integer |
| Storage | Consul |
| Used by | `worker-linehaul-checker` → `LinehaulCheckingWorker` |
| Purpose | Number of upcoming days to scan loads from NetSuite. Only loads with `pickupAppointmentDate` within `[now, now + N days]` are processed. |
| Default | `7` |
| Allowed values | Positive integer |
| Configurable at | Global |

```text
7
```

### 4.3 `apps.workers.linehaul_checker.driver.linehaul_driver_info_slack_channel` — New key

| Field | Value |
| --- | --- |
| Type | String |
| Storage | Consul |
| Used by | `worker-linehaul-checker` → `LinehaulDriverManager` |
| Purpose | Slack channel that receives info notifications: driver skipped (phone already exists), driver created successfully. |
| Default | `staging-logs` |
| Allowed values | Valid Slack channel name |
| Configurable at | Global |

```text
linehaul-driver-info
```

### 4.4 `apps.workers.linehaul_checker.driver.linehaul_driver_exception_slack_channel` — New key

| Field | Value |
| --- | --- |
| Type | String |
| Storage | Consul |
| Used by | `worker-linehaul-checker` → `LinehaulDriverManager` |
| Purpose | Slack channel that receives error notifications when provisioning fails. |
| Default | `staging-logs` |
| Allowed values | Valid Slack channel name |
| Configurable at | Global |

```text
linehaul-driver-exceptions
```

### 4.5 `apps.workers.linehaul_checker.driver.linehaul_driver_sms_welcome_message` — New key

| Field | Value |
| --- | --- |
| Type | String |
| Storage | Consul |
| Used by | `worker-linehaul-checker` → `LinehaulDriverManager` |
| Purpose | SMS template sent to a newly created driver. Supports the `${forgot_password_link}` placeholder. |
| Default | `"Jitsu Linehaul:\n\nSet your password here: ${forgot_password_link}"` |
| Allowed values | Free-text; may contain `${forgot_password_link}` |
| Configurable at | Global |

```text
Jitsu Linehaul:

Set your password here: ${forgot_password_link}
```

### 4.6 `oauth2.reset_password.forgot_password_endpoint` — Existing key

| Field | Value |
| --- | --- |
| Type | String |
| Storage | Consul |
| Used by | `worker-linehaul-checker` → `LinehaulDriverManager` |
| Purpose | Endpoint embedded in the reset-password link of the welcome SMS. |
| Default | `/forgot-password` |
| Allowed values | Valid URL path |
| Configurable at | Global |

```text
/forgot-password
```

---

## 5. Apply / Restart Checklist

| Service | Type | Restart required? | Reason | Order |
| --- | --- | --- | --- | --- |
| `worker-linehaul-checker` | Worker | **Yes** | All Consul keys are read in `setup()` at startup — no hot-reload | 1 |

---

## 6. How to create test sync data from NetSuite

> Use this when you need to test driver provisioning without waiting for the real NetSuite API.
> The worker reads `raw_linehaul_json` directly, bypassing NetSuite entirely.

### Step 1 — Go to the Schedules page

Open: <https://dashboard.staging.gojitsu.com/schedules>

### Step 2 — Find the schedule

Find the schedule named: **`linehaul_checker`**

### Step 3 — Paste content and Run

Paste the following content into the schedule's attributes / payload field, then click **Run**:

```json
{
  "raw_linehaul_json": "[{\"shipmentId\":\"276036\",\"netsuiteRecordData\":{\"lastModified\":\"2026-06-15T03:02:00.000Z\",\"name\":\"PO73583\",\"carrierServiceReview\":\"Satisfactory\",\"closed\":false,\"shipStatus\":\"5. In Transit\",\"shipType\":\"Logistics Movement\",\"linehaulType\":\"First Mile\",\"lane\":\"Riverside, CA to Santa Fe Springs, CA\",\"nonRecurring\":false,\"dispatchNotes\":\"Driver must check in at dock 4\",\"jitsuCustomer\":\"68 Nespresso\",\"jitsuPO\":\"Purchase Order #PO73583\",\"trackStatus\":\"On Time\",\"nsLocation\":\"LAX\",\"p44Id\":\"500391969745\",\"p44URL\":\"https://na12.voc.project44.com/portal/v2/tracking-details/tl/500391969745\",\"p44PublicURL\":\"https://na12.voc.project44.com/portal/v2/public/shipment-details/tl/df077c65-90ca-4589-a4db-d399a900659d\",\"trackOnTime\":\"3\",\"totalPkgs\":42,\"totalWeight\":\"1250 lbs\",\"mode\":\"Truckload\",\"trailerId\":\"TRL-88213\",\"seal\":\"SL-009912\"},\"motorFreightCarrierDetails\":{\"tmsCarrier\":{\"name\":\"GERSON'S LLC TRUCKING\",\"carrierMC\":\"MC-123271\",\"carrierDOT\":\"3162299\"},\"cost\":\"400.00\",\"fuel\":\"65.50\",\"detention\":\"0\",\"detentionComment\":\"\",\"eldDeviceId\":\"ELD-7781\",\"driverName\":\"Carlos Mendez\",\"driverPhone\":\"9097143222\",\"equipment\":\"53' Dry Van\",\"reeferTemp\":\"\",\"carrierLoadId\":\"LD-55021\",\"carrierComments\":\"On schedule\",\"carrierNotesInternal\":\"Preferred carrier for LAX lane\"},\"origin\":{\"shipper\":\"Nespresso\",\"shipAddr1\":\"20820 Krameria Ave.\",\"shipAddr2\":\"Suite 100\",\"shipCity\":\"Riverside\",\"shipState\":\"CA\",\"shipZip\":\"92518\"},\"destination\":{\"consignee\":\"LAX\",\"consigneeAddr1\":\"13943 Maryton Ave\",\"consigneeAddr2\":\"Dock B\",\"consigneeCity\":\"Santa Fe Springs\",\"consigneeState\":\"CA\",\"consigneeZip\":\"90670\"}}]"
}
```

**The sample data above includes:**

| Field | Value |
| --- | --- |
| `shipmentId` | `276036` |
| Carrier | GERSON'S LLC TRUCKING (MC-123271 / DOT-3162299) |
| Driver | Carlos Mendez — `9097143222` |
| Pickup | Riverside, CA 92518 |
| Delivery | Santa Fe Springs, CA 90670 (LAX warehouse) |
| Linehaul type | First Mile |

### Step 4 — Trigger from the schedule

After running, the worker will:

1. Read `raw_linehaul_json` → bypass the NetSuite API
2. Check carrier `GERSON'S LLC TRUCKING` in the DB → create it if missing
3. Check `driverPhone = 9097143222` → create the driver if it does not exist
4. Send a welcome SMS to `9097143222`
5. Log the result to the Slack info/exception channel per config

> **Note**: Make sure `linehaul_driver_provisioning_enabled = true` and that `worker-linehaul-checker` has been restarted before testing.

---

## 7. QA Checklist

### 7.1 Happy path

- [ ] Set `linehaul_driver_provisioning_enabled = true` → restart worker → send a test message to the `linehaul_checker` queue with `raw_linehaul_json` → courier is created in the DB, driver account is created, welcome SMS is sent, Slack info channel receives a message
- [ ] Driver phone already exists in the DB → skip, Slack info channel receives a "phone already exists" message
- [ ] `driverPhone` blank in the payload → the entire flow is skipped, no carrier, no driver, no Slack
- [ ] The same `driverPhone` appears twice in one run → only 1 driver is created, the second is skipped
- [ ] `linehaul_driver_provisioning_enabled = false` → restart worker → send a test message → no carrier/driver is created, linehaul data is still processed normally

### 7.2 Carrier creation

- [ ] `carrierName` has a value and the carrier does not exist in the DB → a Courier record is created with `type = LINEHAUL`
- [ ] `carrierName` has a value and the carrier already exists in the DB → no new record is created, the existing one is reused

### 7.3 Pickup window filter

- [ ] A load with `pickupAppointmentDate` outside `[now, now + window_days]` → is not processed
- [ ] Change `linehaul_driver_provisioning_window_days` → restart → loads within the new window are picked up

### 7.4 Regression

- [ ] When `linehaul_driver_provisioning_enabled = false`: the linehaul record is still synced to the DB as before
- [ ] `linehaul_stop` is still created/updated normally (unaffected by the new feature)

---

## 8. Edge Cases & Gotchas

| Scenario | Behavior |
| --- | --- |
| `driverPhone` blank | Skip everything — no carrier, no driver, no Slack |
| Carrier already exists | Reuse the existing record, do not create a new one |
| Driver phone already exists in the DB | Skip the driver, send a log to the Slack info channel |
| Same phone across multiple loads (1 run) | Process only the first occurrence (dedup by phone) |
| NetSuite API fails | Retry with exponential backoff within the same run window |
| `raw_linehaul_json` present in the message | Bypass NetSuite — parse directly (used for testing) |
| `linehaul_driver_provisioning_enabled = false` | Only disables provisioning, does not disable linehaul sync |
