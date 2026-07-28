# Function: Log-on (Login)

## 1. Description

The login screen authenticates **provisioned** Linehaul drivers (no self sign-up).
Only accounts of type **LINEHAUL** can access the app; DSP, Platform, and IC accounts are
rejected and pointed to their corresponding driver app.

Key characteristics:

- Drivers are provisioned — **no self-registration** on this screen.
- Two entry methods: **email/username** and **phone number**.
- Authentication runs through **Jitsu OAuth2**.
- **No 2FA** and **no offline login**.

---

## 2. Login methods

| Method | Identifier | Notes |
|---|---|---|
| Email/Username (default) | email address **or** username | Single field labeled `Username/Email`. Displayed as typed — no masking, no trimming on display. |
| Phone number | 10-digit US phone | Reached via the **Login with phone number** link. Auto-formatted to `(xxx) xxx-xxxx`. Numeric keyboard only. |

Both methods require a **password** and both trigger the same OAuth2 authentication.

---

## 3. UI elements & placeholders

| Element | Default state |
|---|---|
| `Username/Email` field | Placeholder text: **"Username/Email"** |
| `Password` field | Placeholder text: **"Password"**; input masked as `•••` |
| Password reveal icon | Toggles password between masked `•••` and plain text |
| `Login` button | Primary action; has pressed + loading states |
| `Login with phone number` link | Switches input method to phone |
| `Forgot password` link | Opens Forgot Password screen |
| `Clear cache` link | Opens **Delete App** pop-up |
| Offline banner | Shown only when offline |
| Sign up link | **Not present / not functional** |

---

## 4. Field validation rules

### Username / Email
- **Required** — empty + Login → inline error **"Email or username required"**.
- Accepts letters, spaces, hyphens, apostrophes — e.g. `John O'Brien`, `Mary-Jane`, `user@123` → no format error.
- **Whitespace is trimmed** before submission; whitespace-only input is treated as empty.
- Displayed exactly as typed — no masking.

### Phone number (phone method)
- **Required** — empty + Login → inline error **"Phone Required"**.
- **Max 10 digits** — the field stops accepting input after the 10th digit.
- Must match `(xxx) xxx-xxxx` — otherwise inline error **"Invalid Phone Number"**.
- Auto-formatted to `(xxx) xxx-xxxx` as typed.
- Numeric keyboard only — no alphabetic keyboard.

### Password
- **Required** — empty + Login → inline error **"Password Required"**.
- No complexity rules enforced **at login** (complexity lives in the Reset Password flow).
- Masked by default; toggle via reveal icon.

---

## 5. Authentication flow

```
Driver enters identifier + password → taps Login
        ↓
Jitsu OAuth2 authentication is triggered
        ↓
Backend validates:
  1. credentials are correct
  2. account_type == LINEHAUL          → else reject (§6)
  3. account state allows login        → else reject (§7)
        ↓
Success → navigate to the state-appropriate screen (§7)
Failure → inline error or Authentication Error pop-up (§8)
```

- **Debounce:** rapid/double taps send **only one** auth request; the button is disabled/debounced after the first tap.
- **Loading state:** shown while awaiting the backend response.

---

## 6. Account-type gating

Only **LINEHAUL** accounts may log in. All other account types are rejected.

| Account type | Result | Message |
|---|---|---|
| `LINEHAUL` | ✅ Login proceeds (subject to §7) | — |
| `DSP` | ❌ Rejected | *"Invalid account. Please sign in using your corresponding driver app."* |
| `PLATFORM` | ❌ Rejected | *"Invalid account. Please sign in using your corresponding driver app."* |
| `IC` (Independent Contractor) | ❌ Rejected | *"Invalid account. Please sign in using your corresponding driver app."* |

> ⚠️ **To confirm with PO:** the spec describes two different rejection messages for a DSP account — an **Authentication Error pop-up** with *"Invalid login credentials"* in one place, and the inline *"Invalid account…"* message in another. Confirm which is correct.

---

## 7. Account status → login-destination matrix

Login eligibility and post-login destination depend on the DSP driver state
(see [README §3 — DSP Driver Lifecycle](../README.md#3-dsp-driver-lifecycle--states)).

| Status | Background | In DSP? | Contract | Result | Destination screen |
|---|---|---|---|---|---|
| `CREATED` | `PENDING` | Yes | — | ✅ Login | **3-session screen** |
| `BACKGROUND_APPROVED` | `MANUAL_APPROVED` | Yes | Not signed | ✅ Login | **Contract session screen** |
| `BACKGROUND_APPROVED` | `MANUAL_APPROVED` | Yes | Signed | ✅ Login | **Active assignment screen** |
| `QUIT` | `MANUAL_APPROVED` | Yes | Signed | ✅ Login | **Active screen** |
| `QUIT` | `MANUAL_APPROVED` | **No** (deleted) | — | ❌ Blocked | *"Invalid account…"* → Login screen |
| `SUSPENDED` | `MANUAL_APPROVED` | Yes | — | ✅ Login | **Contract session screen** |
| `CREATED` | `NULL` | No (IC) | — | ❌ Blocked | *"Invalid account…"* → Login screen |

> **Principle:** Only the **deleted** state (`QUIT` + In DSP = No) and **non-LINEHAUL** account types block login. All other states log in; restricting operational actions is handled by app-side gating. **Login ≠ Active.**

---

## 8. Error & message catalog

### Inline field errors

| Trigger | Message |
|---|---|
| Empty username/email | `Email or username required` |
| Empty phone | `Phone Required` |
| Invalid phone format | `Invalid Phone Number` |
| Empty password | `Password Required` |
| Invalid credentials | `Invalid login credentials` |
| Wrong account type / deleted account | `Invalid account. Please sign in using your corresponding driver app.` |

### Authentication Error pop-up

Standard shape:

```
Title:   Authentication error
Content: <see below>
Button:  OK   → returns to Login screen
```

| Scenario | Content |
|---|---|
| Offline login attempt | `Unknown error` |
| Expired / invalid token on open | `Unknown error` |
| Force logout (multi-device) | `Unknown error` |
| Token expires mid-session | `Unknown error` |
| Account locked out | `Account is locked due to many failed attempts. Retry after <N> mins` |


---

## 9. Session management

- **Single-session policy** — logging in on Device B **force-logs-out** Device A. Device A shows the Authentication Error pop-up, then returns to the Login screen.

---

## 10. Token & auto-login

| Scenario | Behavior |
|---|---|
| Valid token stored, app reopened | Login screen **skipped** → navigate directly to Contract/Active screen. No error. |
| Expired/invalid token on open | Authentication Error pop-up (`Unknown error`) → OK → Login screen |
| Token expires mid-session | Next authenticated action → Authentication Error pop-up → OK → Login screen |

- Token lifetime is governed by `default_access_token_expired_duration` (backend config).
- **No offline tokens** — re-authentication is required when the token is invalidated.

---

## 11. Account lockout

Configured via three backend keys:

| Config | Meaning |
|---|---|
| `max_failed_login_attempts` | Number of wrong-password attempts (N) before lockout |
| `login_failed_time_window_sec` | Window within which the N failures must occur; also the retry-after duration surfaced to the user |
| `login_failed_count_ttl_sec` | TTL for the failed-attempt counter |

**Behavior:** entering the wrong password N times within the window temporarily locks the account and shows the Authentication Error pop-up:

```
Title:   Authentication error
Content: Account is locked due to many failed attempts. Retry after <login_failed_time_window_sec> mins
Button:  OK
```

---

## 12. Offline behavior

- **Offline banner** at the bottom of the Login screen: yellow background · white text **"Offline - Not connected"** · upward arrow icon (↑).
- **Login is blocked offline** — any login attempt → Authentication Error pop-up (`Unknown error`) → OK → Login screen.
- There is **no offline login** mode.

---

## 13. Hyperlinks & navigation

| Link / element | Action |
|---|---|
| `Forgot password` | Opens Forgot Password screen |
| `Login with phone number` | Switches input method to phone |
| `Clear cache` | Opens **Delete App** pop-up |
| Sign up | **Not present / not functional** |

---

## 14. Login button behavior

- **Debounce:** double/spam taps → exactly **one** auth request; button disabled/debounced after the first tap.
- **Press & loading:** pressed state on touch; on release the request fires and a loading state is shown until the response returns.
- **Phone keyboard:** tapping the phone number field shows a numeric keyboard only (not alphabetic/full).

---

## 15. Internationalization (i18n)

The login screen is **English-only**, even on a non-English device:

- With device language set to Spanish, all UI text remains English:
  placeholders `Username/Email` / `Password`, button `Login`, links `Login with phone number` / `Forgot password`.
- Validation errors also stay English: `Email or username required`, `Password Required`.

---

## 16. Test setup — Create a Linehaul driver account manually

The login screen has **no self sign-up** — a driver account must already exist before you can test login.
For QA, the fastest way to create a real LINEHAUL account on **staging** is to send a message to the
`linehaul_checker` schedule with a `raw_linehaul_json` payload. This bypasses NetSuite and drives the
same provisioning path the worker uses in production.

> Deep reference (payload fields, courier logic, config keys):
> [auto-create-carrier-and-driver.md §6](./auto-create-carrier-and-driver.md#6-how-to-create-test-sync-data-from-netsuite).
> This section is the QA-focused short version for login testing.

### 16.1 Prerequisites

| Requirement | Why |
|---|---|
| `apps.workers.linehaul_checker.driver.linehaul_driver_provisioning_enabled = true` | Master gate — when `false`, no driver is created |
| `worker-linehaul-checker` **restarted** after the config change | All Consul keys are read once in `setup()` — no hot-reload |
| A `driverPhone` that does **not** already exist in the DB | Existing phone → skipped (dedup), no new account |

### 16.2 Steps

1. Open the Schedules page: <https://dashboard.staging.gojitsu.com/schedules>
2. Find the schedule named **`linehaul_checker`**.
3. Paste a `raw_linehaul_json` payload into the schedule's attributes / payload field, then click **Run**.

```json
{
  "raw_linehaul_json": "[{\"shipmentId\":\"276036\",\"netsuiteRecordData\":{\"lastModified\":\"2026-06-15T03:02:00.000Z\",\"name\":\"PO73583\",\"carrierServiceReview\":\"Satisfactory\",\"closed\":false,\"shipStatus\":\"5. In Transit\",\"shipType\":\"Logistics Movement\",\"linehaulType\":\"First Mile\",\"lane\":\"Riverside, CA to Santa Fe Springs, CA\",\"nonRecurring\":false,\"dispatchNotes\":\"Driver must check in at dock 4\",\"jitsuCustomer\":\"68 Nespresso\",\"jitsuPO\":\"Purchase Order #PO73583\",\"trackStatus\":\"On Time\",\"nsLocation\":\"LAX\",\"p44Id\":\"500391969745\",\"p44URL\":\"https://na12.voc.project44.com/portal/v2/tracking-details/tl/500391969745\",\"p44PublicURL\":\"https://na12.voc.project44.com/portal/v2/public/shipment-details/tl/df077c65-90ca-4589-a4db-d399a900659d\",\"trackOnTime\":\"3\",\"totalPkgs\":42,\"totalWeight\":\"1250 lbs\",\"mode\":\"Truckload\",\"trailerId\":\"TRL-88213\",\"seal\":\"SL-009912\"},\"motorFreightCarrierDetails\":{\"tmsCarrier\":{\"name\":\"GERSON'S LLC TRUCKING\",\"carrierMC\":\"MC-123271\",\"carrierDOT\":\"3162299\"},\"cost\":\"400.00\",\"fuel\":\"65.50\",\"detention\":\"0\",\"detentionComment\":\"\",\"eldDeviceId\":\"ELD-7781\",\"driverName\":\"Carlos Mendez\",\"driverPhone\":\"9097143222\",\"equipment\":\"53' Dry Van\",\"reeferTemp\":\"\",\"carrierLoadId\":\"LD-55021\",\"carrierComments\":\"On schedule\",\"carrierNotesInternal\":\"Preferred carrier for LAX lane\"},\"origin\":{\"shipper\":\"Nespresso\",\"shipAddr1\":\"20820 Krameria Ave.\",\"shipAddr2\":\"Suite 100\",\"shipCity\":\"Riverside\",\"shipState\":\"CA\",\"shipZip\":\"92518\"},\"destination\":{\"consignee\":\"LAX\",\"consigneeAddr1\":\"13943 Maryton Ave\",\"consigneeAddr2\":\"Dock B\",\"consigneeCity\":\"Santa Fe Springs\",\"consigneeState\":\"CA\",\"consigneeZip\":\"90670\"}}]"
}
```

> ⚠️ **Change `driverPhone`** to a new, unused number for each fresh account (an existing phone is
> skipped by dedup). If you add `origin.tracking.pickupAppointmentDate`, keep it **within** the
> provisioning window `[today, today + window_days]` (default `7`) or the load is filtered out.

### 16.3 What gets created

After **Run**, the worker:

1. Reads `raw_linehaul_json` → bypasses the NetSuite API
2. Creates the courier (`GERSON'S LLC TRUCKING`) if it does not already exist — `type = LINEHAUL`
3. Creates the driver if `driverPhone` does not exist, with these states:

   | Field | Value |
   |---|---|
   | `backgroundStatus` | `MANUALLY_APPROVED` |
   | `status` | `BACKGROUND_APPROVED` |
   | Registration phases | `FINISHED` |
   | DL / Birthday | dummy placeholder |

4. Sends a welcome SMS to `driverPhone` containing a reset-password link

This lands the driver at the **`BACKGROUND_APPROVED + MANUAL_APPROVED`** state — a ✅ login state
per the [§7 status matrix](#7-account-status--login-destination-matrix).

### 16.4 Set the password & log in

1. Open the welcome SMS on the test device (or trigger **Forgot password** in the app with the same phone).
2. Follow the reset-password link → set a password.
3. On the login screen, log in with either:
   - **Phone number** `(909) 714-3222` + the new password, or
   - the account's **email/username** + the new password.

### 16.5 Gotchas

| Symptom | Cause |
|---|---|
| No account created, nothing in Slack info channel | `driverPhone` blank, or phone already exists (dedup) |
| Worker ran but no driver | `linehaul_driver_provisioning_enabled = false`, or worker not restarted after config change |
| Load ignored | `pickupAppointmentDate` outside the `[now, now + window_days]` window |
| No welcome SMS received | Invalid / landline phone → flagged as operational exception |
