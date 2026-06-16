# WAT-1906 — Test Cases: Deliverable Address Update Flow

> **Ticket:** [WAT-1906](https://gojitsu.atlassian.net/browse/WAT-1906)  
> **Feature:** Recipient Update Address 1 — Customer Profile & Deliverable Address Logic  
> **Sprint:** 127  
> **Last updated:** 2026-06-16

---

## Business Logic Summary

### Core Lookup Flow

When recipient updates Address 1, the system runs **2 independent lookups in parallel**:

```
Update Address 1
  ├── Lookup CustomerInfo + new Address → CP
  │     ├── EXISTS  → reuse existing CP
  │     └── NOT EXISTS → create new CP
  └── Lookup new Address (street1+street2+state+city+zipcode) → DA
        ├── EXISTS  → reuse existing DA
        └── NOT EXISTS → create new DA
```

**Note:** DA is created/looked up BEFORE CP. Old CP → DA relationship is always preserved.
Therefore `old CP + new DA` is **impossible** — if CP exists, its DA already exists.

### Valid CP × DA Combinations After Address Update

| Combination | Condition |
|---|---|
| **new CP + new DA** | CustomerInfo+new address not in system; new address not in DA |
| **new CP + old DA** | CustomerInfo+new address not in system; new address already has DA |
| **old CP + old DA** | CustomerInfo+new address already has CP (DA always exists for old CP) |
| ~~old CP + new DA~~ | ❌ Impossible — if CP exists, its DA already exists |

### Data Clone Logic (CP)

After address update, the following fields are cloned/overridden from old CP into new/reused CP:

| Field | Behavior |
|---|---|
| `questionnaire_id` | Overridden from old CP → new CP ✅ |
| `access_code_id` | Overridden from old CP → new CP ✅ — ⚠️ **Known Issue**: only the `access_code_id` containing ACCESS_CODE entry only is cloned, NOT the latest one (which may contain entrance/building/call_in codes too) |
| `have_free_roaming_pet` | Cloned from old CP ✅ |
| `preferred_photo_url` | Merged as union (entries from both CPs) ✅ |
| `deliverable_address_id` | **NOT cloned** (fix of WAT-1906 — this was the bug) |

#### access_code_id Clone — Known Issue Detail

Each time recipient adds/changes any code, a new `access_code_id` is generated reflecting all codes at that moment:

```
Recipient adds Access Code = '3'
  → AC-1 created: { ACCESS_CODE: '3' }
  → CP.access_code_id = AC-1

Recipient adds Entrance Code = '1'
  → AC-2 created: { ACCESS_CODE: '3', GATE_CODE: '1' }
  → CP.access_code_id = AC-2  ← current/latest

After Address 1 update → clone logic picks AC-1 (ACCESS_CODE only)
  → New CP.access_code_id = AC-1  ← entrance_code/building_code/call_in_code LOST
```

### Data Clone Logic (DA)

When the new/reused DA has no existing `access_code` / `known_access_codes`:

| Field | Behavior |
|---|---|
| `access_code` (standalone field) | Cloned from old DA ✅ |
| `known_access_codes[access_code]` | Cloned ✅ |
| `known_access_codes[entrance_code]` | **NOT cloned** ⚠️ Known Issue |
| `known_access_codes[building_code]` | **NOT cloned** ⚠️ Known Issue |
| `known_access_codes[call_in_code]` | **NOT cloned** ⚠️ Known Issue |
| `address_type` on DA | **NOT updated** from questionnaire answers (out of scope WAT-1906) |

If reused DA already has `access_code` / `known_access_codes` → **keep existing, do NOT overwrite**.

### `known_access_codes` Structure

```json
{
  "access_code": "116",
  "known_access_codes": [
    { "code": "1", "source": "SHIPMENT", ... },   // entrance_code
    { "code": "2", "source": "SHIPMENT", ... },   // building_code
    { "code": "116", "source": "SHIPMENT", ... }, // access_code (also in standalone field)
    { "code": "4", "source": "SHIPMENT", ... }    // call_in_code
  ]
}
```

### Auto-Correct (DAS) — Sibling Shipments

When Shipment B's address is updated, other shipments sharing the **same CP AND same DA** are:

- **Auto-corrected on Dispatch address display** ✅
- `customer_profile_id` and `deliverable_address_id` are **NOT changed** on sibling shipments
- No `[CustomerProfileDetector] detect-profile` event on sibling shipments

**Eligibility for auto-correct:**

| Condition | Rule |
|---|---|
| Routing problem | ✅ NOT in active routing problem |
| Age | ✅ Created within **60 days** (standard) or **7 days** (`by_date` routing mode) |
| Terminal statuses | ❌ Skipped: `DAMAGED`, `UNDELIVERABLE`, `DROPOFF_SUCCEEDED`, `CANCELLED_*`, `RETURNED_*`, `DISPOSABLE`, `RETURN_DAMAGED` |

**Same DA but different CP → NOT auto-corrected.**

### Key Event for Verification

After address update, find in Dispatch shipment history:

```
[CustomerProfileDetector] detect-profile
  ref.uid  = new/reused CP id
  rel.uid  = new/reused DA id
  fact.old_customer_profile_id = old CP id (source for clone)
```

### Recipient Update Flow (Sequence)

1. Recipient accesses app via shipment `tracking_code` → questionnaire auto-displayed
2. Recipient fills: `address_type`, access codes, `has_pet`, `preferred_photo_url` → saved to **current CP + current DA**
3. Recipient clicks 'Edit Address', enters new Address 1, saves
4. System runs CP + DA lookups in parallel → determines combination
5. Data from old CP cloned into new/reused CP
6. `[CustomerProfileDetector] detect-profile` event fired

### Service Restart Required

| Service | Reason |
|---|---|
| `event-processor-shipment-preprocess` | PR #2988 code change |
| `event-processor-shipment` | PR #2960 — `CustomerProfileDetector.java` |

---

## Test Cases

### CP/DA Combination

#### [TC-01] Update Address 1 → CustomerInfo+new address NOT in system, new address NOT in DA → create new CP + new DA

**Priority:** High  
**Focus:** CP/DA Lookup — new CP + new DA  

**Precondition:**

> Staging env. Shipment A exists with CP-A (CustomerInfo X + Address Y) and DA-A (Address Y).  
> CustomerInfo X + Address Z does NOT exist in customer_profile.  
> Address Z does NOT exist in deliverable_address.  
> Recipient opens Manage Delivery page via Shipment A tracking code → questionnaire is displayed.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record current CP-A.id and CP-A.deliverable_address_id (= DA-A) from MongoDB before any action. | MongoDB: customer_profile — query by shipment's current customer_profile_id | CP-A.id and CP-A.deliverable_address_id = DA-A.id recorded as baseline. Also note DA-A.access_code and DA-A.known_access_codes. |
| 2 | Recipient fills in questionnaire: select address_type, fill access codes (entrance_code, building_code, access_code, call_in_code), set has_pet = true, upload preferred_photo_url. Save each section. | address_type = SINGLE_HOUSE; access_code = '3'; entrance_code = '1'; building_code = '2'; call_in_code = '4'; has_pet = true | Each section saved. Data written to current CP-A (questionnaire_id = Q-B, access_code_id = AC-B, have_free_roaming_pet = true) and DA-A. |
| 3 | Recipient clicks 'Edit Address', enters Address Z (not in system), and saves. | Address Z = a new address not yet in system | Address update accepted. UI displays Address Z. |
| 4 | In Dispatch shipment history, find the '[CustomerProfileDetector] detect-profile' event. Read: ref.uid (new CP), rel.uid (new DA), fact.old_customer_profile_id (old CP). | Dispatch: Shipment history → [CustomerProfileDetector] detect-profile event | Event exists. ref.uid = new CP id. rel.uid = new DA id for Address Z. fact.old_customer_profile_id = CP-A.id. |
| 5 | Query MongoDB customer_profile: new CP (id = ref.uid). Read questionnaire_id, access_code_id, have_free_roaming_pet, preferred_photo_url, deliverable_address_id. | new CP id from step 4 ref.uid | deliverable_address_id = new DA id (= rel.uid). questionnaire_id = Q-B (overridden from old CP). access_code_id = AC-B (overridden from old CP) — ⚠️ Known Issue: only the access_code_id that contains ACCESS_CODE entry only is cloned, NOT the latest access_code_id which may include entrance_code/building_code/call_in_code. If recipient added multiple codes, new CP may have incomplete code set. have_free_roaming_pet = true. preferred_photo_url merged as union (contains entries from old CP). |
| 6 | Query MongoDB deliverable_address: new DA (id = rel.uid from step 4). Verify address matches Address Z and DA document exists. | new DA id from step 4 rel.uid | New DA document exists. address.full matches Address Z. This confirms new CP (ref.uid) is correctly linked to new DA (rel.uid) for the updated address. |
| 7 | Query MongoDB customer_profile: old CP (id = fact.old_customer_profile_id = CP-A). Verify it is unchanged. | CP-A.id from step 4 | CP-A still exists with original field values. Not corrupted by the clone. |

#### [TC-02] Update Address 1 → CustomerInfo+new address NOT in system, new address already has DA → create new CP, reuse existing DA

**Priority:** High  
**Focus:** CP/DA Lookup — new CP + old DA  

**Precondition:**

> Staging env. Shipment A exists with CP-A (CustomerInfo X + Address Y) and DA-A (Address Y).  
> CustomerInfo X + Address Z does NOT exist in customer_profile.  
> Address Z ALREADY exists in deliverable_address with id = DA-Z.  
> Recipient opens Manage Delivery page via Shipment A tracking code → questionnaire is displayed.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-A.id, CP-A.deliverable_address_id (= DA-A). Record DA-Z.id and its fields: access_code, known_access_codes, address_type. | MongoDB: customer_profile, deliverable_address | All baseline values recorded. Note whether DA-Z has access_code / known_access_codes or not. |
| 2 | Recipient fills in questionnaire: address_type, access codes, has_pet = true, preferred_photo_url. Save each section. | address_type = SINGLE_HOUSE; access_code = '3'; entrance_code = '1'; building_code = '2'; call_in_code = '4'; has_pet = true | Data saved to current CP-A (questionnaire_id = Q-B, access_code_id = AC-B, have_free_roaming_pet = true) and DA-A. |
| 3 | Recipient clicks 'Edit Address', enters Address Z (already has DA-Z), and saves. | Address Z = address that already has DA-Z | Address update accepted. |
| 4 | In Dispatch shipment history, find the '[CustomerProfileDetector] detect-profile' event. Read: ref.uid (new CP), rel.uid (DA), fact.old_customer_profile_id (old CP). | Dispatch: [CustomerProfileDetector] detect-profile event | Event exists. ref.uid = new CP id. rel.uid = DA-Z.id (existing DA reused). fact.old_customer_profile_id = CP-A.id. |
| 5 | Query MongoDB customer_profile: new CP (id = ref.uid). Read questionnaire_id, access_code_id, have_free_roaming_pet, preferred_photo_url, deliverable_address_id. | new CP id from step 4 | deliverable_address_id = DA-Z.id (= rel.uid). questionnaire_id = Q-B (overridden from old CP). access_code_id = AC-B (overridden from old CP) — ⚠️ Known Issue: only the access_code_id that contains ACCESS_CODE entry only is cloned, NOT the latest access_code_id which may include entrance_code/building_code/call_in_code. If recipient added multiple codes, new CP may have incomplete code set. have_free_roaming_pet = true. preferred_photo_url merged as union from old CP. |
| 6 | Query MongoDB deliverable_address: reused DA (id = rel.uid from step 4). Verify it is DA-Z and no duplicate was created. | DA-Z id from step 1; rel.uid from step 4 | rel.uid = DA-Z.id (existing DA reused). address.full matches Address Z. Query deliverable_address by full Address Z → exactly 1 document (DA-Z). No duplicate created. New CP (ref.uid) correctly linked to existing DA-Z (rel.uid). |
| 7 | Verify no duplicate DA created for Address Z. | MongoDB: query deliverable_address by full Address Z (street + street2 + city + state + zipcode) | Exactly 1 document = DA-Z. No duplicate. |
| 8 | Query MongoDB customer_profile: old CP (id = fact.old_customer_profile_id = CP-A). Verify it is unchanged. | CP-A.id from step 4 | CP-A still exists with original field values. Not corrupted by the update. |

#### [TC-03] Update Address 1 → CustomerInfo+new address already has CP, address already has DA → reuse both CP and DA

**Priority:** High  
**Focus:** CP/DA Lookup — old CP + old DA  

**Precondition:**

> Staging env. Shipment B exists with CP-B (CustomerInfo X + Address Z) and DA-B (Address Z).  
> CustomerInfo X + Address Y ALREADY exists: CP-A (with Q-A, AC-A). Address Y ALREADY exists: DA-A (may be missing some fields).  
> Recipient opens Manage Delivery page via Shipment B tracking code → questionnaire is displayed.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-B.id and CP-B.deliverable_address_id (= DA-B). Record CP-A.id, CP-A.questionnaire_id (= Q-A), CP-A.access_code_id (= AC-A), CP-A.have_free_roaming_pet, CP-A.preferred_photo_url. Record DA-A.id, DA-A.access_code, DA-A.known_access_codes, DA-A.address_type. | MongoDB: customer_profile, deliverable_address | All baseline values recorded. Note whether DA-A has access_code / known_access_codes or not. |
| 2 | Recipient fills in questionnaire on Shipment B: address_type, access codes (entrance_code, building_code, access_code, call_in_code), has_pet = true, preferred_photo_url. Save each section. | address_type = SINGLE_HOUSE; access_code = '3'; entrance_code = '1'; building_code = '2'; call_in_code = '4'; has_pet = true | Data saved to CP-B (questionnaire_id = Q-B, access_code_id = AC-B, have_free_roaming_pet = true) and DA-B. |
| 3 | Recipient clicks 'Edit Address', enters Address Y (CP-A and DA-A already exist), and saves. | Address Y = address of CP-A and DA-A | Address update accepted. |
| 4 | In Dispatch shipment history, find the '[CustomerProfileDetector] detect-profile' event. Read: ref.uid (reused CP), rel.uid (DA), fact.old_customer_profile_id (old CP). | Dispatch: [CustomerProfileDetector] detect-profile event | Event exists. ref.uid = CP-A.id (existing CP reused). rel.uid = DA-A.id (existing DA reused). fact.old_customer_profile_id = CP-B.id. |
| 5 | Query MongoDB customer_profile: CP-A (id = ref.uid). Read questionnaire_id, access_code_id, have_free_roaming_pet, preferred_photo_url, deliverable_address_id. | CP-A.id from step 4 | deliverable_address_id = DA-A.id (= rel.uid). questionnaire_id = Q-B (overridden from CP-B — recipient does NOT need to re-select). access_code_id = AC-B (overridden from CP-B). have_free_roaming_pet = true. preferred_photo_url merged as union (contains entries from both CP-A and CP-B). |
| 6 | Query MongoDB deliverable_address: reused DA-A (id = rel.uid from step 4). Verify it is DA-A and no new DA was created for Address Y. | DA-A id from step 1; rel.uid from step 4 | rel.uid = DA-A.id (existing DA reused). address.full matches Address Y. Query deliverable_address by full Address Y → exactly 1 document (DA-A). No duplicate created. Reused CP-A (ref.uid) correctly linked to DA-A (rel.uid). |
| 7 | Verify DA-A.address_type after update. | DA-A.id from step 1 | DA-A.address_type is NOT updated by questionnaire answers. DA-A retains original value. Expected behavior — DA address_type sync is out of scope for WAT-1906. |
| 8 | Query MongoDB customer_profile: old CP-B (id = fact.old_customer_profile_id). Verify state. | CP-B.id from step 4 | CP-B still exists in DB. CP-B is orphaned — no shipment references it. Its data is unchanged. |
| 9 | Verify DA-B state after update. | - | DA-B still exists in DB. DA-B is orphaned — no CP references it. Not deleted. |

### DA Access Code Clone

#### [TC-03A] Reuse existing DA that already has access_code and known_access_codes → existing data kept, NOT overwritten

**Priority:** High  
**Focus:** DA — Reuse existing DA: keep existing codes  

**Precondition:**

> Staging env. Recipient updates Address 1 to an address that already has an existing DA in the system.  
> The existing DA (= rel.uid after update) already has:  
> • access_code (standalone field): non-empty (e.g. '116')  
> • known_access_codes: array with entries for multiple code types (entrance_code, building_code, access_code, call_in_code)  
> Old DA (current DA before address update) also has access_code and known_access_codes data.  
> Address update completed. '[CustomerProfileDetector] detect-profile' event visible in Dispatch history.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | From the detect-profile event, read rel.uid (reused DA id). | Dispatch: [CustomerProfileDetector] detect-profile event → rel.uid | rel.uid noted = existing DA id. |
| 2 | Record baseline: existing DA.access_code and all entries in DA.known_access_codes (entrance_code, building_code, access_code, call_in_code values) BEFORE the address update. | MongoDB: deliverable_address by rel.uid — captured before update | All baseline code values recorded. |
| 3 | Query MongoDB deliverable_address: reused DA (id = rel.uid) AFTER the address update. Read access_code and each entry in known_access_codes. | rel.uid from step 1 | access_code (standalone) = baseline value. Unchanged ✅ known_access_codes array = baseline entries. Unchanged ✅ No entries from old DA were appended or overwritten. Existing data is fully preserved. |

#### [TC-03B] New or reused DA has no access_code and no known_access_codes → all 4 code types should be cloned from old DA

**Priority:** High  
**Focus:** DA — Clone codes from old DA when new DA is empty  

**Precondition:**

> Staging env. Recipient updates Address 1 to an address where the new/reused DA (= rel.uid) has:  
> • access_code (standalone field): null or empty  
> • known_access_codes: empty array or null  
> Old DA (current DA before address update) has:  
> • access_code: non-empty value  
> • known_access_codes: entries for all 4 code types (entrance_code, building_code, access_code, call_in_code)  
> Address update completed. '[CustomerProfileDetector] detect-profile' event visible in Dispatch history.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | From the detect-profile event, read rel.uid (new/reused DA id) and fact.old_customer_profile_id (old CP id). | Dispatch: [CustomerProfileDetector] detect-profile event | rel.uid and fact.old_customer_profile_id noted. |
| 2 | Query MongoDB customer_profile: old CP (fact.old_customer_profile_id). Read deliverable_address_id (= old DA id). | old CP id from step 1 | Old DA id noted. |
| 3 | Query MongoDB deliverable_address: old DA. Record access_code and all known_access_codes entries (entrance_code, building_code, access_code, call_in_code values and ids). | old DA id from step 2 | All 4 code type values from old DA recorded as expected clone source. |
| 4 | Query MongoDB deliverable_address: new/reused DA (id = rel.uid). Read access_code (standalone) and all known_access_codes entries after the update. | rel.uid from step 1 | access_code (standalone field): cloned from old DA ✅ known_access_codes entries:   • access_code entry: cloned ✅   • entrance_code entry: cloned ✅ [verify — Known Issue may apply]   • building_code entry: cloned ✅ [verify — Known Issue may apply]   • call_in_code entry: cloned ✅ [verify — Known Issue may apply] Log actual result per entry type. Known Issue: currently only access_code entry is cloned; entrance_code / building_code / call_in_code entries are NOT cloned. |

#### [TC-03C] [Known Issue] New or reused DA has no access_code but has >1 known_access_codes entries → only access_code cloned, other code types lost

**Priority:** Medium  
**Focus:** DA — Known Issue: >1 known_access_codes, only access_code cloned  

**Precondition:**

> Staging env. Recipient updates Address 1 to an address where the new/reused DA (= rel.uid) has:  
> • access_code (standalone field): null or empty  
> • known_access_codes: array with >1 entries (e.g. from geocoder or prior shipment — entrance_code, building_code, or call_in_code entries already present)  
> Old DA (current DA before update) has access_code and known_access_codes with all 4 code types.  
> Address update completed. '[CustomerProfileDetector] detect-profile' event visible in Dispatch history.  
> NOTE: This is a Known Issue as of 2026-06-09. Expected behavior vs actual behavior documented below.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | From the detect-profile event, read rel.uid (new/reused DA id) and fact.old_customer_profile_id. | Dispatch: [CustomerProfileDetector] detect-profile event | rel.uid and fact.old_customer_profile_id noted. |
| 2 | Query MongoDB customer_profile: old CP → get old DA id. Query old DA: record access_code and all known_access_codes entries. | old CP id from step 1 → old DA id | Old DA code values recorded as clone source. |
| 3 | Record baseline of new/reused DA (rel.uid) BEFORE update: confirm access_code = null/empty and known_access_codes has >1 entries. | rel.uid — captured before update | Confirmed: access_code empty, known_access_codes array has >1 existing entries (not from old DA). |
| 4 | Query MongoDB deliverable_address: new/reused DA (rel.uid) AFTER update. Read access_code (standalone) and all known_access_codes entries. | rel.uid from step 1 | EXPECTED (correct behavior): access_code cloned from old DA. known_access_codes: all 4 code type entries (entrance_code, building_code, access_code, call_in_code) cloned from old DA.  ACTUAL [Known Issue]: access_code (standalone) cloned ✅. known_access_codes: only access_code entry cloned. entrance_code / building_code / call_in_code entries from old DA are NOT cloned — lost. Pre-existing entries in known_access_codes remain as-is.  Log actual result. Track as known issue until fixed. |

### CP Access Code Clone

#### [TC-03D] [Known Issue] Recipient adds multiple codes → CP has latest access_code_id with all codes → after Address 1 update, new CP clones access_code_id with ACCESS_CODE only, not latest

**Priority:** High  
**Focus:** CP — Known Issue: access_code_id clone picks first (access_code only) not latest  

**Precondition:**

> Staging env. Recipient has a shipment with current CP.  
> Recipient adds codes in sequence:  
> 1. First adds Access Code only → AC-1 created (contains ACCESS_CODE only)  
> → CP.access_code_id = AC-1  
> 2. Then adds Entrance Code → AC-2 created (contains ACCESS_CODE + GATE_CODE)  
> → CP.access_code_id = AC-2  ← this is the current/latest  
> Recipient then updates Address 1.  
> NOTE: This is a Known Issue. Expected (correct) behavior vs actual behavior documented below.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record baseline CP before address update: access_code_id field and its access_code_map content. | MongoDB: customer_profile → access_code_id; MongoDB: access_code → id = access_code_id | CP.access_code_id = AC-2. AC-2.access_code_map contains both ACCESS_CODE and GATE_CODE entries. |
| 2 | Also record AC-1: query access_code collection for the earlier record that has ACCESS_CODE only. | MongoDB: access_code — find record with only ACCESS_CODE entry for this recipient | AC-1 exists. AC-1.access_code_map contains ACCESS_CODE only (no GATE_CODE, BUILDING_CODE, CALLIN_CODE). |
| 3 | Recipient updates Address 1 and saves. | New Address 1 | Address update accepted. [CustomerProfileDetector] detect-profile event fired. |
| 4 | From detect-profile event, read ref.uid (new/reused CP). Query MongoDB customer_profile: new CP.access_code_id. | ref.uid from [CustomerProfileDetector] detect-profile event | EXPECTED (correct behavior): new CP.access_code_id = AC-2 (latest, contains all codes recipient entered).  ACTUAL [Known Issue]: new CP.access_code_id = AC-1 (ACCESS_CODE only). Clone logic picks the access_code_id that contains ACCESS_CODE entry only, not the latest one. Entrance Code, Building Code, Call-in Code entered by recipient are lost in new CP. |
| 5 | Verify UI on Manage Delivery page after address update: check Codes section. | - | EXPECTED: All codes still displayed (Access Code, Entrance/Gate Code, Building Code, Call-in Code).  ACTUAL [Known Issue]: Only Access Code is shown. Entrance/Gate Code, Building Code, Call-in Code are missing — they were not cloned to new CP. |

### UX

#### [TC-04] Update Address 1 → questionnaire answers and codes retained on UI, no re-selection needed

**Priority:** High  
**Focus:** UX — Recipient does not need to re-select questionnaire after update  

**Precondition:**

> Staging env. Recipient opens Manage Delivery page via shipment tracking code.  
> Questionnaire is displayed. Recipient selects: address_type = Single House, gated = Yes, delivery location = Front Doorstep.  
> Codes filled: Entrance/Gate Code = 1, Building Code = 2, Access Code = 3, Call-in Code = 4.  
> have_free_roaming_pet = true. All sections saved before updating Address 1.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Verify Address Questionnaire section shows the selections just made. | - | Single House selected, Is gated = Yes, Front Doorstep selected. |
| 2 | Verify Codes section shows the values just entered. | - | Entrance/Gate: 1, Building: 2, Access: 3, Call-in: 4. |
| 3 | Recipient clicks 'Edit Address', enters new Address 1, and saves. | New Address 1 (any valid new address) | Address update accepted. UI refreshes. |
| 4 | Verify Address Questionnaire section AFTER address update. | - | Questionnaire answers still displayed (Single House, gated, Front Doorstep). Form is NOT blank or reset. |
| 5 | Verify Codes section AFTER address update. | - | Codes still populated: Entrance/Gate: 1, Building: 2, Access: 3, Call-in: 4. NOT reset to empty. |
| 6 | Verify Delivery Instructions AFTER address update. | - | 'I have free roaming pets' is still checked. |

### Multi-Shipment & DAS Correct

#### [TC-05] 2 shipments same CP + same DA → update Address 1 on Shipment B → Shipment A is auto corrected to new address

**Priority:** High  
**Focus:** Multi-Shipment — same CP+DA, update one shipment  

**Precondition:**

> Staging env. Shipment A and Shipment B share the same customer_profile_id = CP-X and deliverable_address_id = DA-X.  
> Both shipments are in pre-geocoded or GEOCODED status (not yet DELIVERED).  
> Recipient opens Manage Delivery page via Shipment B tracking code.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-X.id and CP-X.deliverable_address_id (= DA-X). Record customer_profile_id for both Shipment A and B from PostgreSQL. | PostgreSQL: shipments; MongoDB: customer_profile | Both shipments reference CP-X → DA-X. Baseline recorded. |
| 2 | Recipient updates Address 1 of Shipment B to Address Z and saves. | Address Z = new address | Address update accepted for Shipment B. |
| 3 | Find '[CustomerProfileDetector] detect-profile' event for Shipment B in Dispatch history. Read ref.uid (new CP), rel.uid (new DA), fact.old_customer_profile_id. | Dispatch: Shipment B history | Event exists. ref.uid = new CP for Shipment B. rel.uid = new DA for Address Z. fact.old_customer_profile_id = CP-X. |
| 4 | Verify Shipment B: query PostgreSQL customer_profile_id and MongoDB CP.deliverable_address_id. | - | Shipment B.customer_profile_id = ref.uid (new CP). New CP.deliverable_address_id = rel.uid (new DA for Address Z). |
| 5 | Verify Shipment A on Dispatch: check delivery address displayed after Shipment B's update. | Shipment A id | Shipment A delivery address IS auto corrected to Address Z on Dispatch view (same DA was updated). Shipment A shows new address. |
| 6 | Query PostgreSQL: customer_profile_id of Shipment A. Query MongoDB: CP document — read id and deliverable_address_id. | - | Shipment A.customer_profile_id still = CP-X (NOT changed). CP-X.deliverable_address_id still = DA-X (NOT changed). Only the address display is auto corrected — CP and DA ids are not updated. |

#### [TC-05A] Upload new shipment with invalid address → address auto corrected to valid address by DAS

**Priority:** High  
**Focus:** DAS Correct — Invalid address on new shipment upload  

**Precondition:**

> Staging env. A new shipment is uploaded with an address that is invalid or unrecognizable as-is (e.g. street number slightly off, missing unit info that DAS can resolve).  
> DAS correct flow is active.  
> Shipment processes through worker and CustomerProfileDetector.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Upload new shipment with invalid/correctable address. Note the raw address from the upload file. | e.g. address = '640 Up To Ave, Anchorage AK 99501' (invalid) → DAS corrects to valid address | Shipment created. Initial address shows as uploaded. |
| 2 | Wait for DAS correct to process. Check Dispatch: Dropoff Info address displayed. | - | Dispatch shows the DAS-corrected valid address (not the original invalid one). Shipment status = GEOCODED. |
| 3 | Find '[CustomerProfileDetector] detect-profile' event in Dispatch shipment history. Read ref.uid (CP) and rel.uid (DA). | Dispatch: [CustomerProfileDetector] detect-profile event | Event exists. ref.uid = CP for corrected address. rel.uid = DA for corrected address. |
| 4 | Query MongoDB customer_profile: CP (ref.uid). Read deliverable_address_id and address fields. | ref.uid from step 3 | CP.deliverable_address_id = rel.uid (DA of corrected address). CP address fields reflect the corrected valid address. |
| 5 | Query MongoDB deliverable_address: DA (rel.uid). Read address.full. | rel.uid from step 3 | DA.address.full = corrected valid address. Matches what Dispatch displays. |

#### [TC-05B] 2 shipments same DA but different CP → update Address 1 on Shipment B → Shipment A is NOT auto corrected

**Priority:** High  
**Focus:** DAS Correct — Same DA, different CP: update 1 does NOT propagate to other  

**Precondition:**

> Staging env. Shipment A and Shipment B share the same deliverable_address_id = DA-X BUT have different customer_profile_ids:  
> • Shipment A → CP-A → DA-X  
> • Shipment B → CP-B → DA-X  
> Both shipments are in active status (not DELIVERED).  
> Recipient opens Manage Delivery page via Shipment B tracking code.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-A.id, CP-B.id, DA-X.id. Record Dispatch address displayed for Shipment A before update. | PostgreSQL: shipments; MongoDB: customer_profile | Baseline recorded. Both CPs reference DA-X. Shipment A address noted. |
| 2 | Recipient updates Address 1 of Shipment B to Address Z and saves. | Address Z = new address | Address update accepted for Shipment B. |
| 3 | Find '[CustomerProfileDetector] detect-profile' event for Shipment B. Read ref.uid and rel.uid. | Dispatch: Shipment B [CustomerProfileDetector] detect-profile event | Shipment B now references new/reused CP (ref.uid) with new DA for Address Z (rel.uid). |
| 4 | Verify Shipment A on Dispatch: check delivery address displayed after Shipment B's update. | Shipment A id | Shipment A delivery address is NOT auto corrected. Still shows original address (DA-X address). Shipment A is unaffected because it has a different CP from Shipment B. |
| 5 | Query PostgreSQL: Shipment A.customer_profile_id. Query MongoDB: CP-A.deliverable_address_id. | - | Shipment A.customer_profile_id = CP-A (unchanged). CP-A.deliverable_address_id = DA-X (unchanged). No auto correction applied to Shipment A. |

#### [TC-06] Shipment A COMPLETED + Shipment B active share same CP → update B does not affect A

**Priority:** High  
**Focus:** Multi-Shipment — Completed shipment not modified  

**Precondition:**

> Staging env. Shipment A status = DELIVERED/COMPLETED. Shipment B status = active.  
> Both share customer_profile_id = CP-X.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-X.id and CP-X.deliverable_address_id (= DA-X) for Shipment A. | MongoDB: customer_profile | Baseline recorded. |
| 2 | Recipient updates Address 1 of Shipment B and saves. | New Address 1 | Update accepted for Shipment B. |
| 3 | Query PostgreSQL: customer_profile_id of Shipment A. Query MongoDB: CP document — read id and deliverable_address_id. | - | Shipment A still references CP-X with deliverable_address_id = DA-X. COMPLETED shipment is NOT modified. |

#### [TC-07] 2 shipments same CP + DA — Shipment A geocoding in progress, Shipment B updates Address 1 → Shipment A auto corrected, no CP/DA event on A

**Priority:** High  
**Focus:** Concurrency — Geocoding vs Address Update  

**Precondition:**

> Staging env. Shipment A and Shipment B share the same customer_profile_id = CP-X and deliverable_address_id = DA-X.  
> Shipment A is currently being geocoded (status: in geocoding process).  
> Shipment A is eligible for auto-correct: not in active routing problem, created within 60 days, not in terminal status.  
> Recipient opens Manage Delivery page via Shipment B tracking code.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-X.id, DA-X.id. Confirm Shipment A is in geocoding state. Record Shipment A current address. | PostgreSQL: shipments; Dispatch: Shipment A status | Shipment A is geocoding. Both shipments reference CP-X → DA-X. Baseline recorded. |
| 2 | While Shipment A is geocoding, recipient updates Address 1 of Shipment B to Address Z and saves. | Address Z = new address | Address update accepted for Shipment B. |
| 3 | Find '[CustomerProfileDetector] detect-profile' event for Shipment B in Dispatch history. | Dispatch: Shipment B history | Event exists for Shipment B. ref.uid = new/reused CP for Address Z. rel.uid = new/reused DA for Address Z. |
| 4 | Check Dispatch shipment history for Shipment A: look for any '[CustomerProfileDetector] detect-profile' event triggered by the address update. | Dispatch: Shipment A history | NO '[CustomerProfileDetector] detect-profile' event on Shipment A triggered by this update. Shipment A's CP and DA are NOT updated via profile detection. |
| 5 | Wait for Shipment A geocoding to complete. Verify Dispatch address displayed for Shipment A. | Shipment A id | Shipment A delivery address IS auto corrected to Address Z on Dispatch view (same CP+DA as Shipment B → eligible for auto-correct). Geocoding completes without error. |
| 6 | Query PostgreSQL: Shipment A.customer_profile_id. Query MongoDB: CP-X.deliverable_address_id after update. | - | Shipment A.customer_profile_id = CP-X (unchanged). CP-X.deliverable_address_id = DA-X (unchanged). Auto correct updated address display only — CP and DA ids on Shipment A are NOT modified. |

#### [TC-08] DAS correct — shipments sharing same DA and same CP auto-corrected → verify eligibility criteria (NOTE: overlaps with TC-05)

**Priority:** High  
**Focus:** DAS Correct — sibling shipments auto-corrected, CP+DA not updated  

**Precondition:**

> Staging env. Multiple shipments (S1, S2, S3) share the same customer_profile_id = CP-X AND deliverable_address_id = DA-X.  
> Eligibility for auto-correct per shipment:  
> ✅ Not in active routing problem  
> ✅ Created within 60 days (standard) or 7 days (routing mode = by_date)  
> ❌ Terminal statuses are skipped: DAMAGED, UNDELIVERABLE, DROPOFF_SUCCEEDED, CANCELLED_*, RETURNED_*, DISPOSABLE, RETURN_DAMAGED  
> S2 is eligible (active, within 60 days). S3 is in a terminal status (e.g. DROPOFF_SUCCEEDED).  
> Recipient updates Address 1 on S1.  
> NOTE: Core auto-correct behavior (same CP + same DA) is covered in TC-05. This TC focuses on eligibility filtering.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-X.id, DA-X.id. Record customer_profile_id and status for S1, S2, S3. | PostgreSQL: shipments; MongoDB: customer_profile | S1, S2 reference CP-X → DA-X (active). S3 references CP-X → DA-X but has terminal status (e.g. DROPOFF_SUCCEEDED). Baseline recorded. |
| 2 | Recipient updates Address 1 on S1 to Address Z and saves. | Address Z = new address | Address update accepted for S1. |
| 3 | Verify Dispatch address for S2 after DAS correct. | S2 id | S2 delivery address IS auto corrected to Address Z. S2 is eligible (active, within 60 days, not terminal status). |
| 4 | Verify Dispatch address for S3 after DAS correct. | S3 id | S3 delivery address is NOT auto corrected. S3 is in terminal status (DROPOFF_SUCCEEDED) → skipped by eligibility check. |
| 5 | Query PostgreSQL + MongoDB: S2 and S3 customer_profile_id and CP.deliverable_address_id after update. | - | S2.customer_profile_id = CP-X (unchanged). S3.customer_profile_id = CP-X (unchanged). Both CPs still reference DA-X. Only address display is auto corrected for eligible shipments — CP and DA ids are NOT updated. |
| 6 | Test eligibility boundary: verify a shipment created >60 days ago (same CP+DA) is NOT auto corrected. | Shipment created >60 days ago with same CP-X and DA-X | Shipment is NOT auto corrected. Exceeds 60-day eligibility window. |

### Regression

#### [TC-14] Recipient fills questionnaire and codes but does NOT update Address 1 → CP and DA unchanged

**Priority:** Medium  
**Focus:** Regression — No-op when Address 1 not updated  

**Precondition:**

> Staging env. Recipient opens Manage Delivery page via tracking code. Shipment has CP-X and DA-X.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Record CP-X.id and CP-X.deliverable_address_id from MongoDB. | - | Baseline recorded. |
| 2 | Recipient fills in questionnaire, updates Codes (Gate Code = '99'), saves sections. Does NOT click 'Edit Address'. | Gate Code = 99 | Questionnaire and code updates saved successfully. |
| 3 | Query PostgreSQL: customer_profile_id of the shipment. Query MongoDB: CP document — read id and deliverable_address_id. | - | customer_profile_id = CP-X (unchanged). deliverable_address_id = DA-X (unchanged). No new CP or DA created. |

#### [TC-15] Update Address 1 multiple times including returning to original address → system reuses existing CP+DA correctly

**Priority:** High  
**Focus:** Regression — Multiple updates, return to old address reuses existing CP+DA  

**Precondition:**

> Staging env. Shipment A has CP-A (CustomerInfo X + Address Y) and DA-A (Address Y).  
> Address Z and Address W are new addresses not yet in the system.  

| Step | Action | Test Data | Expected Result |
|------|--------|-----------|-----------------|
| 1 | Update Address 1 to Address Z (no existing CP, no existing DA for Address Z). | Address Z | New CP-Z and DA-Z created. Query PostgreSQL + MongoDB: Shipment references CP-Z with deliverable_address_id = DA-Z. |
| 2 | Update Address 1 to Address W (no existing CP, no existing DA for Address W). | Address W | New CP-W and DA-W created. Shipment references CP-W with deliverable_address_id = DA-W. |
| 3 | Update Address 1 back to Address Y (CP-A and DA-A already exist in the system). | Address Y = original address | System reuses CP-A. Query PostgreSQL + MongoDB: customer_profile_id = CP-A.id, deliverable_address_id = DA-A.id. No new CP created. |
| 4 | Verify count of CP documents for CustomerInfo X + Address Y in MongoDB. | MongoDB: customer_profile query by CustomerInfo X + Address Y | Exactly 1 document = CP-A. No duplicate CP created. |

---

## Related Tickets

| Ticket | Type | Description |
|---|---|---|
| [WAT-1917](https://gojitsu.atlassian.net/browse/WAT-1917) | Spike | Root cause analysis — old DA retained in new CP |
| [WAT-1906](https://gojitsu.atlassian.net/browse/WAT-1906) | Bug Fix | Remove `deliverable_address_id` from `cloneProfileToNewProfile()` |
| [ENG-9786](https://gojitsu.atlassian.net/browse/ENG-9786) | Story | Recipient Update Address 1 feature |

## Related PRs

| PR | Repo | Change |
|---|---|---|
| #2960 | worker | Remove `deliverable_address_id` from clone map + audit |
| #2988 | worker | `cloneDeliverableAddressData` — DA access code clone logic |
