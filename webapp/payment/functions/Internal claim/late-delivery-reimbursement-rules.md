# Late Delivery Reimbursement Rules

**Epic:** WAT-1912 — Internal Claims v1.1  
**Feature:** Late Delivery Credit (Late OTD Reimbursement)  
**Last updated:** 2026-06-29

---

## Overview

When a shipment is delivered late (actual departure timestamp exceeds OTD deadline), the system can
automatically calculate a **late delivery credit** on top of the standard claim payout. This credit
is applied separately from the loss/damage claim amount.

The feature is **opt-in per client** and requires configuration in three layers:

1. `claims_client_policy` (Postgres) — enable the feature and define reimbursement tiers
2. `financial_preferences` (MongoDB) — link the client to an invoice pricing model
3. `finance_model` (MongoDB) — define the `base_formula` that calculates the delivery fee

---

## How It Works (Execution Flow)

```
processBatch()
  └── evaluateClaim()          → ClaimDecision (APPROVED / REJECTED / PENDING_REVIEW)
  └── isApprovedWithPayout()   → must be true (status=APPROVED AND approvedCreditAmount > 0)
  └── applyPayout()            → applies threshold / payout rule
  └── applyGlobalCap()         → caps approved amount
  └── applyLateDeliveryCredit() ← late delivery credit calculated here
```

`applyLateDeliveryCredit()` is called for **every** claim, but returns early unless all conditions
are met (see Gate Conditions below).

---

## Gate Conditions — All Must Pass

The late delivery credit is **only calculated** when ALL of the following are true:

| # | Condition | Source |
|---|-----------|--------|
| 1 | `result.status == "APPROVED"` | Claim must be auto-approved (not PENDING_REVIEW, not REJECTED) |
| 2 | `policy.lateReimburseAmount == "true"` | `claims_client_policy.late_reimburse_amount` = `'true'` (string) |
| 3 | `policy.lateDeliveryPayoutRule != null` | `claims_client_policy.late_delivery_payout_rule` is configured |
| 4 | Delivery fee exists for the shipment | `price_tickets.base` must exist in Postgres |
| 5 | `minutesLate > 0` | `stop.actualDepartureTs > shipment.otdLatestTs` |
| 6 | Rule tier matches `minutesLate` | Configured rule must cover the actual lateness in minutes |

**If any gate fails → `late_delivery_credit = null` (not shown in response).**

---

## Configuration: `claims_client_policy` (Postgres)

Table: `claims_client_policy`

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `late_reimburse_amount` | `varchar` | **YES** | Must be the string `'true'` to enable the feature. `null` or any other value disables it. |
| `late_delivery_payout_rule` | `text` | **YES** | Rule string defining reimbursement tiers (see formats below) |

Other columns used for context (not specific to late delivery):

| Column | Description |
|--------|-------------|
| `client_id` | Client being configured |
| `submission_window_days` | Claim submission window (affects D1 deny rule) |
| `loss_damage_threshold_pct` | Loss/damage threshold (not related to late delivery) |
| `loss_damage_payout_rule` | Loss/damage payout rule (not related to late delivery) |

### Example — Enable Late Delivery Credit for a Client

```sql
UPDATE claims_client_policy
SET
    late_reimburse_amount    = 'true',
    late_delivery_payout_rule = '1-15 min late: 50% of delivery fee; 16+ min late: 80% of delivery fee'
WHERE client_id = <CLIENT_ID>;
```

---

## Rule String Format

### Format A — Flat Rule

Applies a fixed percentage regardless of how late the shipment is.

```
<PCT>% of delivery fee
```

**Example:**

```
50% of delivery fee
```

→ Credit = `delivery_fee × 0.50`

### Format B — Tiered Rule (Recommended)

Applies different percentages depending on how many minutes late.

```
<MIN>-<MAX> min late: <PCT>% of delivery fee; <MIN>+ min late: <PCT>% of delivery fee
```

**Examples:**

```
1-15 min late: 50% of delivery fee; 16+ min late: 80% of delivery fee
```

```
1-15 min late: 25% of delivery fee; 16-60 min late: 50% of delivery fee; 61+ min late: 100% of delivery fee
```

### Tier Syntax Rules

| Segment | Meaning |
|---------|---------|
| `1-15 min late` | Matches if `1 ≤ minutesLate ≤ 15` |
| `16+ min late` | Matches if `minutesLate ≥ 16` (unbounded upper) |
| `16-60 min late` | Matches if `16 ≤ minutesLate ≤ 60` |

- Tiers are separated by `;`
- Tiers are evaluated in order; first match wins
- If `minutesLate` does not match any tier → credit = `null` (not applied)
- Rule string must be ≤ 500 characters

### Common Mistakes

| Mistake | Result |
|---------|--------|
| `late_reimburse_amount = null` | Feature disabled entirely, no credit shown |
| `late_reimburse_amount = 'TRUE'` | Works — comparison is case-insensitive |
| Rule only covers `1-15 min late`, shipment is 344 min late | No tier matches → credit = null |
| Rule uses `DROP_OFF` instead of `delivery fee` | Regex won't match → credit = null |
| `minutesLate = 0` (on-time or missing timestamps) | Returns early, no credit |

---

## Configuration: Delivery Fee Source

The delivery fee used for late delivery credit calculation is:

```
price_tickets.base  (Postgres)
```

This value is calculated at the time of pricing by `ShipmentPricingManager` using:

### Layer 1: `financial_preferences` (MongoDB)

```
Collection: financial_preferences
Filter:     { subject_id: <CLIENT_ID>, scope: "INVOICE", subject: "CLIENT" }
Key field:  model  →  references finance_model.id
```

### Layer 2: `finance_model` (MongoDB)

```
Collection: finance_model
Filter:     { id: <model_id_from_financial_preferences>, scope: "INVOICE" }
Key field:  base_formula  →  expression evaluated at pricing time
```

**`base_formula` examples:**

```js
// Flat rate
"6.25"

// Region-based lookup
"by_region[region]"

// Conditional
"(volume < 1238 ? 5.5 : 6.3)"
```

The formula result is stored as `price_tickets.base` when the shipment is priced.

### Required Setup to Show Delivery Fee

| Step | What to configure | Where |
|------|-------------------|-------|
| 1 | Create or link a `finance_model` document with `scope=INVOICE` and a valid `base_formula` | MongoDB `finance_model` |
| 2 | Create a `financial_preferences` document: `{ subject_id: <client_id>, subject: "CLIENT", scope: "INVOICE", model: "<finance_model.id>" }` | MongoDB `financial_preferences` |
| 3 | Ensure shipment is priced (pricing pipeline runs and writes `price_tickets.base`) | Postgres `price_tickets` |

If `price_tickets.base` is `null` or missing → `applyLateDeliveryCredit()` returns early, no credit shown.

---

## How Minutes Late Is Calculated

```java
// ClaimBatchProcessor.java — calculateMinutesLate()
long diffMs = dropoffStop.actualDepartureTs.getMillis() - shipment.otdLatestTs.getMillis();
return diffMs / (60 * 1000);
```

| Field | DB Column | Table |
|-------|-----------|-------|
| `actualDepartureTs` | `actual_departure_ts` | `stops` (where `type = 'deliverShipment'`) |
| `otdLatestTs` | `otd_latest_ts` | `shipments` |

**Note:** DB stores stop type as `deliverShipment`, not `DROP_OFF`. The API exposes it as `DROP_OFF`.

If either timestamp is `null` → `minutesLate = 0` → no credit.

### Seconds Are Truncated (Floor to Whole Minutes)

`minutesLate` only keeps the **whole minutes** — the seconds portion is always discarded
(integer division floors the result). The truncated value is what feeds the tier matching and the
Late Delivery Credit calculation.

| Actual lateness vs OTD | `minutesLate` used |
|------------------------|--------------------|
| 15m 01s late | **15** |
| 15m 59s late | **15** |
| 16m 00s late | **16** |
| 0m 59s late | **0** → no credit |

**Example:** A shipment delivered **15m 01s** after OTD is treated as **15 min late** (not 16),
so it matches the `1-15 min late` tier when calculating the Late Delivery Credit.

---

## API Response Fields

When late delivery credit is calculated successfully, the claim result includes:

| Field | Description |
|-------|-------------|
| `delivery_fee` | Value of `price_tickets.base` (e.g., `20.00`) |
| `late_delivery_credit` | Calculated credit amount (e.g., `16.00` = 80% of $20) |

These fields are **not shown** when:
- The claim is PENDING_REVIEW or REJECTED
- `late_reimburse_amount` is not `'true'`
- No delivery fee found in `price_tickets`
- Shipment was on time
- Rule has no matching tier for the actual lateness

---

## Claim Status Requirements

Late delivery credit only applies to **auto-approved** claims. It does NOT apply to:

| Status | Late Credit? | Reason |
|--------|-------------|--------|
| `APPROVED` (auto) | **YES** (if all gates pass) | Only eligible status |
| `PENDING_REVIEW` | NO | `isApprovedWithPayout()` = false; `applyLateDeliveryCredit()` returns early |
| `REJECTED` | NO | Same as above |
| `APPROVED` with `approvedCreditAmount = 0` (Below Threshold) | NO | `isApprovedWithPayout()` returns false when amount = 0 |

**Important:** A claim that is PENDING_REVIEW due to late OTD (no auto-decision rule matched) will
**never** receive an auto-calculated late delivery credit. Manual review is required for such cases.

---

## Test Data Reference

### Client 455 — Fully Configured (Reference)

```
claims_client_policy:
  late_reimburse_amount    = 'true'
  late_delivery_payout_rule = '1-15 min late: 50% of delivery fee; 16+ min late: 80% of delivery fee'

financial_preferences:
  subject_id = 455
  scope      = INVOICE
  model      = "0139ea40-9551-4989-b195-c78235b41ffd"

finance_model (id = 0139ea40-...):
  base_formula  = by_region[region]  (default $20.00)
```

### Client 646 — Partially Configured (Missing `late_reimburse_amount`)

```
claims_client_policy:
  late_reimburse_amount    = null   ← MUST SET TO 'true'
  late_delivery_payout_rule = '1-15 min late: 90% of delivery fee'  ← Only covers 1-15 min

financial_preferences:
  subject_id = 646
  scope      = INVOICE
  model      = "0139ea40-9551-4989-b195-c78235b41ffd"

price_tickets.base = 20.00  (confirmed for shipment 70444353)
```

**To fix client 646:**

```sql
UPDATE claims_client_policy
SET
    late_reimburse_amount    = 'true',
    late_delivery_payout_rule = '1-15 min late: 90% of delivery fee; 16+ min late: 80% of delivery fee'
WHERE client_id = 646;
```

---

## Test Cases

### TC01 — Delivery fee shown, credit calculated (happy path)

- **Precondition:** `late_reimburse_amount = 'true'`, rule covers the lateness, `price_tickets.base` exists, claim is APPROVED
- **Input:** Shipment delivered 20 min late, delivery fee = $20, rule = `16+ min late: 80%`
- **Expected:** `delivery_fee = 20.00`, `late_delivery_credit = 16.00`

### TC02 — `late_reimburse_amount` not set

- **Precondition:** `late_reimburse_amount = null`
- **Expected:** `delivery_fee` and `late_delivery_credit` not in response

### TC03 — Rule does not cover actual lateness

- **Precondition:** Rule = `1-15 min late: 90%`, shipment is 344 min late
- **Expected:** `delivery_fee` shown, `late_delivery_credit = null`

### TC04 — Claim is PENDING_REVIEW (late OTD, no auto-approve rule)

- **Precondition:** All policy configured correctly, but claim returns PENDING_REVIEW
- **Expected:** Neither `delivery_fee` nor `late_delivery_credit` shown (credit skipped entirely)

### TC05 — On-time delivery

- **Precondition:** `actualDepartureTs ≤ otdLatestTs`
- **Expected:** `minutesLate ≤ 0` → no credit

### TC06 — Missing `price_tickets.base`

- **Precondition:** `financial_preferences` or `finance_model` not configured for client
- **Expected:** `fee = null` → `applyLateDeliveryCredit()` returns early, no credit

### TC07 — Below Threshold claim (approved but $0 payout)

- **Precondition:** Claim auto-approved but `approvedCreditAmount = 0.0` due to threshold suppression
- **Expected:** `isApprovedWithPayout() = false` → `applyLateDeliveryCredit()` not reached

---

## Relevant Code Locations

| File | Method | Purpose |
|------|--------|---------|
| `ClaimBatchProcessor.java:161` | `applyLateDeliveryCredit()` | Main gate logic and credit assignment |
| `ClaimBatchProcessor.java:191` | `calculateMinutesLate()` | Computes lateness from timestamps |
| `ClaimThresholdService.java:150` | `calculateLateDeliveryCredit(rule, fee, minutesLate)` | Tiered rule parsing and credit calculation |
| `ClaimThresholdService.java:176` | `TIER_PATTERN` | Regex for parsing tier rule segments |
| `ClaimThresholdService.java:129` | `calculateLateDeliveryCredit(rule, fee)` | Flat rule fallback (2-arg overload) |
| `ClaimsClientPolicy.java:28` | `lateReimburseAmount` | Policy model field mapping |
| `ClaimsClientPolicy.java:31` | `lateDeliveryPayoutRule` | Policy model field mapping |
