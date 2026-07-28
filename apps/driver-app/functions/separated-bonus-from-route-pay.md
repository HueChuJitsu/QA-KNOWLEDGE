# Separated Bonus from Route Base Pay

## 1. Description

Bonus pay is separated from route base pay in both the payout processing pipeline and the driver-app UI for IC drivers. When the `separate_bonus_pay_and_route_pay` feature flag is enabled per region, the system creates distinct transactions for route base pay, route bonus, and driver bonus instead of a single combined transaction. On the UI side, drivers see a separate "Bonus: +$X.XX" chip distinct from base route pay. DSP/3P payout and display flows are unchanged. The feature is off by default and requires explicit enablement per region.

**Actors:** IC drivers, Ops (per-region flag enablement)

---

## 2. Business Flow

### Payout Pipeline (Backend)

**Flag OFF (default behaviour)**
1. IC driver completes assignment with bonus
2. `TXCreator.processAssignment` creates a single combined transaction (route pay + bonus merged)
3. Tracking event emitted — no separate `driver_bonus` fact

**Flag ON**
1. IC driver completes assignment with bonus
2. `TXCreator.processAssignment` creates **3 separate transactions**:
   - `TYPE_ROUTE_DRIVING` — route base pay
   - `OUTVOICE_BONUS` with note `[label] Route Bonus`
   - `OUTVOICE_BONUS` with note `[label] Driver Bonus`
3. Tracking event emits a new `driver_bonus` fact: `receivedAmount − tourCost − routeBonus`

**DSP/3P** — unchanged regardless of flag state.

### Driver App UI (driver-app-api)

**Flag OFF**
- Total pay shown as before; no "Bonus" chip
- Driver bonus chip label updated: `+$X bonus` → `Driver bonus: +$X` *(immediate, no flag needed)*
- Fuel subsidy no longer bundled into displayed price *(immediate, no flag needed)*

**Flag ON**
- `routeBasePay` shown as primary amount
- Tooltip breakdown: Total / Route base / Bonus
- "Bonus: +$X.XX" chip visible for route bonus
- `driver_bonus_area` slot added to acceptance screen HTML template (Mongo)

**Entry condition:** IC driver assignment with non-zero bonus  
**Exit condition:** Transactions created / UI chips rendered per flag state

---

## 3. Spec / Rules

### Feature Flag
- **Name:** `separate_bonus_pay_and_route_pay`  
- **Scope:** Region (APP_FEATURES, AP_DEFAULT)  
- **Default:** `false` — must be explicitly enabled per region via AppFeaturesManager  
- **Constant:** `AppFeature.Features.SEPARATE_BONUS_PAY_FEATURE` (dispatch-bizlogic)

### Transaction Rules (flag ON, IC only)
| Transaction | Type | Note |
|---|---|---|
| Route base pay | `TYPE_ROUTE_DRIVING` | — |
| Route bonus | `OUTVOICE_BONUS` | `[label] Route Bonus` |
| Driver bonus | `OUTVOICE_BONUS` | `[label] Driver Bonus` |

- If `bonus == 0` → only `TYPE_ROUTE_DRIVING` created, no bonus rows
- If `bonus < 0` → deduction folded into base pay, no separate bonus transaction
- `tourCost` and `bonus` on `Assignment` are nullable — all calculation paths guard for null

### UI Rules (driver-app-api)
- `BonusUtils.separateBonus()` checks flag and splits `tourCost` into `routeBasePay + routeBonusPay`
- Decorators affected: `ActiveAssignmentDecorator`, `ReservableAssignmentDecorator`, `OpenAssignmentDecorator`
- `LayoutGroup.Subsidy` shows "Bonus: +$X.XX" chip for route bonus when flag ON
- Mongo HTML templates for acceptance screen must have `route_bonus_area` and `driver_bonus_area` slots — excludes DSP templates (manual post-deploy step)
- DSP/courier assignments → no bonus separation, flag ignored

---

## 4. QA / Test Notes

### Happy Path — Flag OFF
- IC driver completes route with assignment bonus → single combined transaction created; no separate bonus rows
- Driver app shows old combined total; no "Bonus" chip; driver bonus label reads "Driver bonus: +$X"

### Happy Path — Flag ON
- IC driver completes route with assignment bonus → 3 transactions created: `TYPE_ROUTE_DRIVING`, `OUTVOICE_BONUS` "Route Bonus", `OUTVOICE_BONUS` "Driver Bonus"
- Driver app shows `routeBasePay` as primary with tooltip breakdown; "Bonus: +$X.XX" chip visible
- Active route card: bonus breakdown visible
- Reservable route card: bonus breakdown visible

### Complete Separation Flow
1. Assignment Completed
   ↓
2. Calculate Total Pay (route pay + bonuses)
   ↓
3. Check if separateBonusPay = true
   ↓
4. IF TRUE:
   ├─→ Create Transaction: TYPE_ROUTE_DRIVING (base route pay)
   ├─→ Create Transaction: OUTVOICE_BONUS (route bonus, if > 0)
   └─→ Create Transaction: OUTVOICE_BONUS (driver bonus, if IC and > 0)
   ↓
5. IF FALSE:
   └─→ Create Single Transaction: TYPE_ROUTE_DRIVING (total pay)
   ↓
6. Display in UI with separate chips/notes

### Edge Cases
- **Zero bonus (flag ON):** Only `TYPE_ROUTE_DRIVING` transaction; no bonus rows; no "Bonus" chip
- **Negative bonus (flag ON):** Deduction folded into base pay; no separate bonus transaction
- **Null tourCost/bonus:** System handles gracefully without exception (null-safety guards in place)
- **DSP/3P driver:** Transaction and display unchanged regardless of flag state
- **Tracking event (flag ON):** Verify `driver_bonus` fact = `receivedAmount − tourCost − routeBonus`
- **Fuel subsidy:** No longer inflates price field regardless of flag (immediate change)

### Config / DB Checks
- `item_metadata` updated: `{ scope: "APP_FEATURES", owner: "AP_DEFAULT" }` → `separate_bonus_pay_and_route_pay` added to `values`
- Mongo HTML acceptance screen templates (non-DSP) have `route_bonus_area` and `driver_bonus_area` slots

### Service Restart
| Service | Type | Reason |
|---|---|---|
| `event-processors` | Worker | New AppFeaturesManager + MetadataDAO wired at startup; clientCache removed |
| `driver-app-api` | API | New code with BonusUtils wired at startup |

### Ticket Related
https://gojitsu.atlassian.net/browse/WAT-1840
https://gojitsu.atlassian.net/browse/MOB-2380
https://gojitsu.atlassian.net/browse/MOB-2373