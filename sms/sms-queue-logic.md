# Recipient SMS — Send Pipelines (Immediate vs Queue)

There are **2 pipelines** for sending recipient SMS. Which pipeline an SMS takes depends on the **SMS type** and the **worker/event** that handles it. Both pipelines ultimately go through `SMSComposer.sendRecipientNotificationSMS` → resolve the opt-in phone (IMPL-194), so opt-in applies to both.

## 1. Pipeline A — Send IMMEDIATELY (real-time)

**Path:** Stop status event → `ShipmentStatusUpdater` (reads `sms_trigger.yaml`) → `SMSComposer.triggerMessaging` → publish straight to the `communication` exchange → `CommunicationViaSMSWorker` → provider.

**Characteristics:** sent as soon as the event occurs; logged to `sms_log` / `communication_logs` immediately.

### SMS sent IMMEDIATELY (NOT through sms_queue)

| SMS type | Trigger |
| --- | --- |
| `INFORM_RECIPIENT_PICKUP_SUCCEEDED` | Pickup succeeded |
| `INFORM_RECIPIENT_PICKUP_SUCCEEDED_ID_REQUIRED` | Pickup succeeded + ID required |
| `INFORM_RECIPIENT_PICKUP_SUCCEEDED_SIGNATURE_REQUIRED` | Pickup succeeded + signature required |
| `INFORM_RECIPIENT_PICKUP_FAILED` | Pickup failed |
| `INFORM_RECIPIENT_DROPOFF_SUCCEEDED` | Dropoff succeeded |
| `INFORM_RECIPIENT_DROPOFF_SUCCEEDED_REASON` | Dropoff succeeded (left at...) |
| `INFORM_RECIPIENT_DROPOFF_FAILED` | Dropoff failed (generic) |
| `INFORM_RECIPIENT_DROPOFF_FAILED_REASON` | Dropoff failed (reason ∈ config list) |
| `INFORM_RECIPIENT_DROPOFF_FAILED_UNRECOVERABLE` | Dropoff failed (unrecoverable reason) |
| `INFORM_RECIPIENT_NEXT_IN_LINE` | Next stop in line |
| `INFORM_RECIPIENT_PICKUP_REMINDER` | Pickup reminder |
| `INFORM_RECIPIENT_COMING_SOON` | Driver arriving soon |
| `CTIA_REPLY_START` | First-contact opt-in welcome (`SMSOptInWelcomeHandler`) |
| `INFORM_RECIPIENT_FEEDBACK_REQUEST` | Post-delivery feedback (`IncomingCommunicationViaSMSWorker`) |

## 2. Pipeline B — Through `sms_queue`, then sent on a SCHEDULE (scheduled / batched)

**Path:** Event → `SmsQueueWorker.addSmsQueue` → `sms_queue` collection (status `NOT_CONFIRMED`) → **`SMSQueueProcessSchedule`** (runs on a schedule, per region) → `SMSComposer.sendRecipientNotificationSMS` → status `SENT`.

**Characteristics:** sent with a **delay** (batched by schedule + region); batched to give the recipient a chance to provide an access code / delivery instruction first. Tracked in `sms_queue` (NOT_CONFIRMED → SENT) before reaching `sms_log`.

**Trigger events** (`SmsQueueMessageType`): `FAILED_STOP`, `DRIVER_ACCEPT_ROUTE`, `INBOUND_SCAN`, `DELIVERABLE_ADDRESS_INCIDENT`.

### SMS that go THROUGH sms_queue (`ALL_SMS_QUEUE_TYPES` in `SmsQueueUtils`)

| SMS type | Group |
| --- | --- |
| `ACCESS_CODE` (legacy) | Access code |
| `CONFIRM_ACCESS_CODE` | Access code |
| `CONFIRM_ADDRESS` | Address confirm |
| `CONFIRM_CORRECTED_ADDRESS` | Address confirm |
| `CONFIRM_UNIT` | Address confirm |
| `CONFIRM_ACCESS_CODE_MDU` | Access code (MDU) |
| `CONFIRM_ACCESS_CODE_MDU_NO_INSTRUCTION` | Access code (MDU) |
| `FAILED_DELIVERY_CONFIRM_ACCESS_CODE` | Access code |
| `INFORM_RECIPIENT_DAS_ADDRESS_AUTO_CORRECTED` | DAS address |
| `INFORM_RECIPIENT_DROPOFF_FAILED_NO_ACCESS_CODE` | Reason-specific dropoff failed |
| `INFORM_RECIPIENT_DROPOFF_FAILED_WRONG_ACCESS_CODE` | Reason-specific dropoff failed |
| `INFORM_RECIPIENT_DROPOFF_FAILED_ACCESS_BLOCKED` | Reason-specific dropoff failed |
| `INFORM_RECIPIENT_DROPOFF_FAILED_ADDRESS_INVALID` | Reason-specific dropoff failed |
| `INFORM_RECIPIENT_DROPOFF_FAILED_COMMUNICATION_ISSUE` | Reason-specific dropoff failed |
| `INFORM_RECIPIENT_DROPOFF_FAILED_ACCESS_DENIED_BY_DOORMAN` | Reason-specific dropoff failed |
| `INFORM_RECIPIENT_DROPOFF_FAILED_BUSINESS_CLOSED` | Reason-specific dropoff failed |

## 3. How to tell them apart

| Criterion | Pipeline A (IMMEDIATE) | Pipeline B (QUEUE) |
| --- | --- | --- |
| SMS type | General status notifications | Access-code / address-confirm / reason-specific dropoff-failed / DAS |
| Trigger | Stop status change (`sms_trigger.yaml`) | `FAILED_STOP`, `DRIVER_ACCEPT_ROUTE`, `INBOUND_SCAN`, `DELIVERABLE_ADDRESS_INCIDENT` |
| Worker | `ShipmentStatusUpdater` | `SmsQueueWorker` → `SMSQueueProcessSchedule` |
| Timing | Real-time | Scheduled (per region) — delayed |
| Tracking | `sms_log` / `communication_logs` immediately | `sms_queue` (NOT_CONFIRMED → SENT), then `sms_log` |

## 4. sms_queue — Status lifecycle & processing logic

### 4.1 Status (`SmsQueue.Status`)

| Status | Meaning |
| --- | --- |
| `NOT_CONFIRMED` | Just enqueued — eligible to send on the next schedule run |
| `OPENED` | Recipient opened the link — re-processed (reminder) after ≥ 5 minutes |
| `CONFIRMED` | Recipient confirmed the access code |
| `SENT` | Sent (after `SMSQueueProcessSchedule` processed it) |
| `IGNORED` | Shipment is `DISPOSABLE` → skipped, not sent |

### 4.2 Enqueue — `SmsQueueWorker.addSmsQueue`

On receiving an `SmsQueueMessage` (one of the 4 triggers), the worker finds a `FAILED` dropoff stop whose `reason ∈ STOP_REASON_MAP`, passes the suppression checks, then `addSmsQueue` → creates an `sms_queue` entry with `status=NOT_CONFIRMED`, `sent_count=0`, plus `region`, `reason`, `type`, `shipment_id`.

**Suppression (do NOT enqueue):**

- `DRIVER_ACCEPT_ROUTE` + reason `ACCESS_DENIED_BY_DOORMAN` → skip.
- (reason `NO_ACCESS_CODE` **or** `shipment.dropoffNote` is non-empty) **and** `customer_shipment.updateTs` > the stop-fail time (`predictedDepartureTs`) → skip (recipient already provided an instruction after the failure). Note: this only applies when a `customer_shipment` record exists; no record → no suppression.

### 4.3 Scheduled send — `SMSQueueProcessSchedule.process` (runs per region, on a schedule)

1. **Query NOT_CONFIRMED:** `listToSendSMSByTypes(region, fromCreatedDate = startTime − 1 day, [NOT_CONFIRMED], ALL_SMS_QUEUE_TYPES)` — only entries from the **last 1 day**.
2. **Query OPENED:** `listOpenSmSByTypes(region, now − 5 minutes, ALL_SMS_QUEUE_TYPES)` — `OPENED` entries older than 5 minutes (reminder resend).
3. **Merge** the 2 lists, filter **`sentCount < 2`** (max **2 sends** per shipment), dedupe by `shipmentId`.
4. Per shipment: acquire a **Redis lock** `sms-queue-<shipmentUID>` (wait 5s, hold 30s) → `processSendSMS`.

### 4.4 `processSendSMS` — conditions & behavior

- Shipment `DISPOSABLE` → mark **IGNORED**, do not send.
- Resolve access code: if `dropoffAccessCode` is empty → look up `CustomerProfile → DeliverableAddress.accessCode`. If found → `isNoAccessCode=false`.
- Reason `NO_ACCESS_CODE` but an access code now exists → change reason → `INVALID_ACCESS_CODE`.
- Build `recipient_url` = `<trackingBaseUrl>/delivery?code=<code>&reason=<reason>`; `smsType = SMS_TYPE.fromString(queue.type)`.
- Test-phone filter: if config `only_enable_phone_number` is set, only send to numbers in the list.
- Send via **`SMSComposer.sendRecipientNotificationSMS`** (resolves the opt-in phone — opt-in > manifest).
- Update `status=SENT`, `sent_count++`, publish tracking event `SHIPMENT.COMMUNICATION.sms-queue`.

### 4.5 Timing & retry summary

- An entry is only processed within a **1-day window** from creation.
- Sent at most **2 times** (`sentCount < 2`).
- `OPENED` is re-processed after **5 minutes**.
- Depends on **`SMSQueueProcessSchedule` running for that region** — if the schedule doesn't run (deploy/config gap), the entry is stuck at `NOT_CONFIRMED`.

## 5. Important notes

- A single **dropoff failure** can trigger **both pipelines**:
  - `ShipmentStatusUpdater` → `INFORM_RECIPIENT_DROPOFF_FAILED` / `_REASON` — sent IMMEDIATELY.
  - `SmsQueueWorker` → reason-specific (e.g. `..._ACCESS_DENIED_BY_DOORMAN`) — goes to the QUEUE, sent on schedule.
- An sms_queue SMS showing status `NOT_CONFIRMED` + `sent_count=0` → **waiting for the schedule to send**, NOT an error. If it stays that way for a long time → `SMSQueueProcessSchedule` is not running for that region (deploy/config gap).
- Suppression for Pipeline B (not enqueued): `DRIVER_ACCEPT_ROUTE` + `ACCESS_DENIED_BY_DOORMAN`; or a delivery instruction (`dropoffNote` / `customer_shipment.updateTs`) updated after the stop failed.
- Opt-in phone: both pipelines resolve via `SMSComposer.resolveRecipientPhone` (opt-in > manifest).
