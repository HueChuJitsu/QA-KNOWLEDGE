# DL Scan Details Storage (Check-in Flow)

> Documentation for a Dataorch/Dispatch function.
> Source: [MOB-2764](https://gojitsu.atlassian.net/browse/MOB-2764) — [BE] Store extracted driver license details on DL scan events (check-in flow)
> Service: `dataorch`

## 1. Description

Persists the driver license (DL) details extracted during the OB (Outbound) team's driver check-in scan step, storing them alongside the scan event as a durable record for verification, audit, and downstream use.

**Actors:**
- **OB team member** (warehouse staff/clerk/manager, dispatcher, admin) — scans the driver's DL or manually approves a driver at check-in; the extracted fields are captured automatically.
- **Super-admin** — the only role able to read the decrypted DL details afterwards.

Previously only the scan event itself was recorded — the extracted field values were discarded. Storing them provides a verifiable record tied to the specific scan, supports future verification logic (e.g. matching scanned name/DOB against the driver profile), and gives an audit trail if a DL's validity is later questioned.

⚠️ DL data is sensitive PII — it follows the team's existing encryption and access-control patterns.

## 2. Business Flow

**Write flow — DL scan (verify):**
1. OB team member scans the driver's license during check-in.
2. The app extracts the DL fields and sends them in an optional `dl_details` object on `POST /outbounds/drivers/{driver_id}/verify` (application/json). The driver is validated by the `license` field in the request body; `dl_details` is stored alongside but not used for validation.
3. If `dl_details` contains ≥1 field → one `scan_event_dl_details` row is created (Mongo collection), linked 1:1 to the scan event.
4. Partial extraction → present fields stored, missing fields = null.
5. `dl_details` not sent / empty / unparseable → no row written; scan event still recorded as before (server logs a warning and continues — **check-in is never blocked**).

**Write flow — manual approve (entering DL number):**
1. The OB team member manually enters the driver's DL number in the Outbound app instead of scanning.
2. This flow also uses the `POST /outbounds/drivers/{driver_id}/verify` endpoint, but the `dl_details` JSON is empty — so **no row is created** in `scan_event_dl_details`. The scan event is still recorded as before.

**Write flow — manual approve (`/manually-approve`, photo upload):**
1. When the scanner is unavailable, the OB team member manually approves the driver via `POST /outbounds/drivers/manually-approve` (multipart/form-data).
2. A new optional form field `dl_details` carries the raw JSON object as text (one line, not double-encoded).
3. Same row-creation rules as the verify flow apply.

**Read flow:**
1. A super-admin calls `GET /outbounds/scan-dl-details/{id}` (the record's own id, **not** the scan event id).
2. The decrypted record is returned. Any other role → `403`.

**Exit conditions / guarantees:**
- Feature is fully **additive** — an old outbound app that sends nothing behaves exactly as before (no DL row, scan event recorded, check-in never blocked).

## 3. Spec / Rules

**Configuration:**
- Config `enable_require_id_scan = true` in the `item_metadata` collection / owner = `RG_{region}` will apply for all driver types.
- For IC drivers: just need to config `enable_require_id_scan = true` in `item_metadata` / owner = `DR_{driver_id}`.

**Storage:** new Mongo collection `scan_event_dl_details`, 1:1 with the DL scan event (if `dl_details` has data).

**`dl_details` schema — all fields optional:**

| Field | Encrypted at rest? |
|---|---|
| `first_name`, `last_name`, `full_name` | Yes |
| `date_of_birth` | Yes |
| `address_street`, `address_city`, `address_state`, `address_zip` | Yes |
| `dl_number` | Yes |
| `expiration_date`, `issuing_state` | No (plaintext, kept queryable) |
| `scan_event_id` | link to the `id` from `driver_license_scan_event` collection |

**Endpoints & roles:**

| Endpoint | Format | Roles |
|---|---|---|
| `POST /outbounds/drivers/{driver_id}/verify` | JSON, `dl_details` nested in body | admin, super-admin, dispatcher, warehouse-manager, warehouse-clerk, warehouse-staff |
| `POST /outbounds/drivers/manually-approve` | multipart, `dl_details` = raw JSON string | same as above |
| `GET /outbounds/scan-dl-details/{id}` | returns decrypted record | **super-admin only** (else 403) |

**Rules:**
- `dl_details` with ≥1 field → 1 row created, linked to the scan event.
- Manual approve by entering the DL number in the Outbound app does NOT create a `scan_event_dl_details` row — the `dl_details` JSON is empty in this case.
- Partial extraction → present fields stored, missing fields null.
- Omitted / empty / unparseable → no row; scan event still recorded (warn log, flow never blocked).
- Manual-approve `dl_details` must be the raw JSON object on one line — do NOT double-encode/quote it (a double-encoded string fails to parse; server warns and skips, does not block approval).
- Date format: backend stores the exact string sent (String fields) — convention is `yyyy-MM-dd`.
- Field names are snake_case; FE sends only what was extracted.

**Sample payloads:**

Verify — full extraction:
```json
{
  "license": "D1234567",
  "warehouse_id": 42,
  "verify_only": false,
  "dl_details": {
    "first_name": "John",
    "last_name": "Doe",
    "full_name": "John Doe",
    "date_of_birth": "1990-01-02",
    "address_street": "123 Main St",
    "address_city": "Oakland",
    "address_state": "CA",
    "address_zip": "94607",
    "dl_number": "D1234567",
    "expiration_date": "2030-01-01",
    "issuing_state": "CA"
  }
}
```
Verify - Manual approve by entering DL number

```json
{
  "license": "D1234567",
  "warehouse_id": 42,
  "verify_only": false,
  "dl_details": {
    
  }
}
```

Verify — partial extraction:
```json
{ "license": "D1234567", "warehouse_id": 42, "dl_details": { "full_name": "John Doe", "dl_number": "D1234567", "expiration_date": "2030-01-01" } }
```

Verify — old app / extraction failed (omit `dl_details`):
```json
{ "license": "D1234567", "warehouse_id": 42 }
```

Manually-approve (multipart, `dl_details` as raw JSON text):
```bash
curl -X POST "$BASE/outbounds/drivers/manually-approve" \
  -H "Authorization: Bearer <token>" \
  -F "driver_id=12345" \
  -F "warehouse_id=42" \
  -F "reason=Scanner unavailable" \
  -F "file=@/path/dl.jpg" \
  -F 'dl_details={"full_name":"John Doe","dl_number":"D1234567","date_of_birth":"1990-01-02","issuing_state":"CA"}'
```

## 4. QA / Test notes

**Required config:** `enable_require_id_scan = true` in `item_metadata` / owner = `RG_{region}`.

**Happy cases:**
- Scan succeeds + full `dl_details` → one `scan_event_dl_details` row, linked to the corresponding scan event, all sent fields present.
- Every scan / manual approve generates a NEW record — even for the same driver, in the same warehouse, on the same day, or on a rescan — as long as the `dl_details` JSON has data (identical or different from previous scans). Each scan event gets its own `scan_event_dl_details` row (1:1); records are never merged or overwritten.
- Manual-approve with `dl_details` → row written under the same rules.
- `GET /outbounds/scan-dl-details/{id}` as super-admin → decrypted values returned.

**Edge cases:**
- Partial `dl_details` (e.g. address not parsed) → row still written; present fields populated, missing fields null.
- `dl_details` omitted / empty / unparseable → NO row written; scan event still recorded per existing behavior; check-in not blocked.
- Manual-approve with a double-encoded `dl_details` string → server warns and skips it; approval not blocked; no row written.
- Manual approve by entering the DL number in the Outbound app → NO row written (`dl_details` JSON is empty in this case); approval proceeds normally.
- Old app version (never sends `dl_details`) → behaves exactly as before the feature.
- Non-super-admin role calls `GET /outbounds/scan-dl-details/{id}` → `403`.

**Things to watch when testing:**
- **DB check (Mongo `scan_event_dl_details`)**: sensitive fields (name, DOB, address, dl_number) must NOT be human-readable (encrypted at rest); `expiration_date` / `issuing_state` are plaintext.
- The read endpoint takes the **record's own id**, not the scan event id — easy to mix up.
- Deploy info: service `dataorch`, feature branch `MOB-2764-Store-Extracted-DL`, release branch `release/mob-2764`.

**Sample test data (Staging):**
```json
{"first_name":"Thuy","last_name":"Tran","full_name":"Thuy Tran","date_of_birth":"2000-10-10","address_street":"1 Dummy Street","address_city":"San Leandro","address_state":"CA","address_zip":"10002","dl_number":"11111111","expiration_date":"2033-06-30","issuing_state":"CA"}
```

**Verification status:** Verified on Staging — PASSED (2026-07-06). All four staging scenarios passed: row written + linked + encrypted; partial data handled with nulls; manual-approve creates row; super-admin-only read enforced.
