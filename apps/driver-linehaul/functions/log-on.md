# Function: Log-on (Login)

> **App:** Linehaul Driver App · **Origin ticket:** [MOB-2550](https://gojitsu.atlassian.net/browse/MOB-2550)
> **AC set:** ENG-TC-20097 → ENG-TC-20156 · **Version:** Linehaul Driver App v2.8.0
> **Last updated:** July 2026

Login screen of the Linehaul Driver App. This document synthesizes the full acceptance
criteria and test-case suite for the login feature into a single detailed spec.

---

## Table of Contents

1. [Description](#1-description)
2. [Login methods](#2-login-methods)
3. [UI elements & placeholders](#3-ui-elements--placeholders)
4. [Field validation rules](#4-field-validation-rules)
5. [Authentication flow](#5-authentication-flow)
6. [Account-type gating](#6-account-type-gating)
7. [Account status → login-destination matrix](#7-account-status--login-destination-matrix)
8. [Error & message catalog](#8-error--message-catalog)
9. [Session management](#9-session-management)
10. [Token & auto-login](#10-token--auto-login)
11. [Account lockout](#11-account-lockout)
12. [Offline behavior](#12-offline-behavior)
13. [Hyperlinks & navigation](#13-hyperlinks--navigation)
14. [Login button behavior](#14-login-button-behavior)
15. [Internationalization (i18n)](#15-internationalization-i18n)
16. [Discrepancies & open questions](#16-discrepancies--open-questions)
17. [Test case suite](#17-test-case-suite)

---

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

> Refs: ENG-TC-20097 (email), ENG-TC-20098 (username), ENG-TC-20099 (phone), ENG-TC-20114 (method switch).

---

## 3. UI elements & placeholders

| Element | Default state |
|---|---|
| `Username/Email` field | Placeholder text: **"Username/Email"** (ENG-TC-20110) |
| `Password` field | Placeholder text: **"Password"**; input masked as `•••` (ENG-TC-20111) |
| Password reveal icon | Toggles password between masked `•••` and plain text (ENG-TC-20109) |
| `Login` button | Primary action; has pressed + loading states (ENG-TC-20141) |
| `Login with phone number` link | Switches input method to phone (ENG-TC-20114) |
| `Forgot password` link | Opens Forgot Password screen (ENG-TC-20113) |
| `Clear cache` link | Opens **Delete App** pop-up (ENG-TC-20115) |
| Offline banner | Shown only when offline (ENG-TC-20119) |
| Sign up link | **Not present / not functional** (ENG-TC-20112) |

---

## 4. Field validation rules

### Username / Email
- **Required** — empty + Login → inline error **"Email or username required"** (ENG-TC-20103).
- Accepts letters, spaces, hyphens, apostrophes — e.g. `John O'Brien`, `Mary-Jane`, `user@123` → no format error (ENG-TC-20108).
- **Whitespace is trimmed** before submission; whitespace-only input is treated as empty (ENG-TC-20143).
- Displayed exactly as typed — no masking (ENG-TC-20097/20098).

### Phone number (phone method)
- **Required** — empty + Login → inline error **"Phone Required"** (ENG-TC-20104).
- **Max 10 digits** — the field stops accepting input after the 10th digit (ENG-TC-20105).
- Must match `(xxx) xxx-xxxx` — otherwise inline error **"Invalid Phone Number"** (ENG-TC-20106).
- Auto-formatted to `(xxx) xxx-xxxx` as typed (ENG-TC-20099).
- Numeric keyboard only — no alphabetic keyboard (ENG-TC-20142).

### Password
- **Required** — empty + Login → inline error **"Password Required"** (ENG-TC-20107).
- No complexity rules enforced **at login** (complexity lives in the Reset Password flow).
- Masked by default; toggle via reveal icon (ENG-TC-20109).

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

- **Debounce:** rapid/double taps send **only one** auth request; the button is disabled/debounced after the first tap (ENG-TC-20140).
- **Loading state:** shown while awaiting the backend response (ENG-TC-20141).

> Refs (E2E): ENG-TC-20136 (email full flow), ENG-TC-20137 (phone full flow), ENG-TC-20138 (DSP rejected), ENG-TC-20139 (token auto-login).

---

## 6. Account-type gating

Only **LINEHAUL** accounts may log in. All other account types are rejected.

| Account type | Result | Message | Ref |
|---|---|---|---|
| `LINEHAUL` | ✅ Login proceeds (subject to §7) | — | 20097–20099 |
| `DSP` | ❌ Rejected | *"Invalid account. Please sign in using your corresponding driver app."* | 20100 |
| `PLATFORM` | ❌ Rejected | *"Invalid account. Please sign in using your corresponding driver app."* | 20101 |
| `IC` (Independent Contractor) | ❌ Rejected | *"Invalid account. Please sign in using your corresponding driver app."* | 20102 |

> ⚠️ **Discrepancy:** the E2E DSP case (ENG-TC-20138) shows the rejection via an **Authentication Error pop-up** with content *"Invalid login credentials"*, whereas the unit-level DSP case (ENG-TC-20100) shows the inline *"Invalid account…"* message. See [§16](#16-discrepancies--open-questions).

---

## 7. Account status → login-destination matrix

Login eligibility and post-login destination depend on the DSP driver state
(see [README §3 — DSP Driver Lifecycle](../README.md#3-dsp-driver-lifecycle--states)).

| Status | Background | In DSP? | Contract | Result | Destination screen | Ref |
|---|---|---|---|---|---|---|
| `CREATED` | `PENDING` | Yes | — | ✅ Login | **3-session screen** | 20127 |
| `BACKGROUND_APPROVED` | `MANUAL_APPROVED` | Yes | Not signed | ✅ Login | **Contract session screen** | 20097–20099 |
| `BACKGROUND_APPROVED` | `MANUAL_APPROVED` | Yes | Signed | ✅ Login | **Active assignment screen** | 20128, 20136 |
| `QUIT` | `MANUAL_APPROVED` | Yes | Signed | ✅ Login | **Active screen** | 20129 |
| `QUIT` | `MANUAL_APPROVED` | **No** (deleted) | — | ❌ Blocked | *"Invalid account…"* → Login screen | 20130 |
| `SUSPENDED` | `MANUAL_APPROVED` | Yes | — | ✅ Login | **Contract session screen** | 20131 |
| `CREATED` | `NULL` | No (IC) | — | ❌ Blocked | *"Invalid account…"* → Login screen | 20132 |

> **Principle:** Only the **deleted** state (`QUIT` + In DSP = No) and **non-LINEHAUL** account types block login. All other states log in; restricting operational actions is handled by app-side gating. **Login ≠ Active.**

---

## 8. Error & message catalog

### Inline field errors

| Trigger | Message | Ref |
|---|---|---|
| Empty username/email | `Email or username required` | 20103 |
| Empty phone | `Phone Required` | 20104 |
| Invalid phone format | `Invalid Phone Number` | 20106 |
| Empty password | `Password Required` | 20107 |
| Invalid credentials | `Invalid login credentials` | 20133–20135 |
| Wrong account type | `Invalid account. Please sign in using your corresponding driver app.` | 20100–20102, 20130, 20132 |

### Authentication Error pop-up

Standard shape:

```
Title:   Authentication error
Content: <see below>
Button:  OK   → returns to Login screen
```

| Scenario | Content | Ref |
|---|---|---|
| Offline login attempt | `Unknown error` | 20120 |
| Expired / invalid token on open | `Unknown error` | 20122 |
| Force logout (multi-device) | `Unknown error` | 20123 |
| Token expires mid-session | `Unknown error` | 20124 |
| Account locked out | `Account is locked due to many failed attempts. Retry after <N> mins` | 20125 |

> ⚠️ The content *"Unknown error"* comes straight from the AC and reads like a placeholder.
> Confirm the intended user-facing copy with PO. See [§16](#16-discrepancies--open-questions).

---

## 9. Session management

- **Single-session policy** — logging in on Device B **force-logs-out** Device A. Device A shows the Authentication Error pop-up, then returns to the Login screen (ENG-TC-20123).
- The multi-device force-logout case is **not automated** (manual test).

---

## 10. Token & auto-login

| Scenario | Behavior | Ref |
|---|---|---|
| Valid token stored, app reopened | Login screen **skipped** → navigate directly to Contract/Active screen. No error. | 20121, 20139 |
| Expired/invalid token on open | Authentication Error pop-up (`Unknown error`) → OK → Login screen | 20122 |
| Token expires mid-session | Next authenticated action → Authentication Error pop-up → OK → Login screen | 20124 |

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

**Behavior (ENG-TC-20125):** entering the wrong password N times within the window temporarily locks the account and shows the Authentication Error pop-up:

```
Title:   Authentication error
Content: Account is locked due to many failed attempts. Retry after <login_failed_time_window_sec> mins
Button:  OK
```

---

## 12. Offline behavior

- **Offline banner** at the bottom of the Login screen: yellow background · white text **"Offline - Not connected"** · upward arrow icon (↑) (ENG-TC-20119).
- **Login is blocked offline** — any login attempt → Authentication Error pop-up (`Unknown error`) → OK → Login screen (ENG-TC-20120).
- There is **no offline login** mode.

---

## 13. Hyperlinks & navigation

| Link / element | Action | Ref |
|---|---|---|
| `Forgot password` | Opens Forgot Password screen | 20113 |
| `Login with phone number` | Switches input method to phone | 20114 |
| `Clear cache` | Opens **Delete App** pop-up | 20115 |
| Sign up | **Not present / not functional** | 20112 |

---

## 14. Login button behavior

- **Debounce (ENG-TC-20140):** double/spam taps → exactly **one** auth request; button disabled/debounced after the first tap.
- **Press & loading (ENG-TC-20141):** pressed state on touch; on release the request fires and a loading state is shown until the response returns.

---

## 15. Internationalization (i18n)

The login screen is **English-only**, even on a non-English device (ENG-TC-20156):

- With device language set to Spanish, all UI text remains English:
  placeholders `Username/Email` / `Password`, button `Login`, links `Login with phone number` / `Forgot password`.
- Validation errors also stay English: `Email or username required`, `Password Required`.

---

## 16. Discrepancies & open questions

| # | Item | Detail | Owner |
|---|---|---|---|
| 1 | DSP rejection message | Inline *"Invalid account…"* (20100) vs pop-up *"Invalid login credentials"* (20138 E2E) — which is correct? | PO |
| 2 | `Unknown error` copy | Auth Error pop-up content is literally `Unknown error` across 20120/20122/20123/20124 — placeholder or final copy? | PO |
| 3 | Username pattern vs email/phone | An earlier spec pattern `^[a-zA-Z\s'-]+$` rejects digits/`@`, conflicting with email/phone login. 20108 accepts `user@123`. Confirm final field rules. | PO |
| 4 | Auth provider | This suite says **Jitsu OAuth2**; earlier spec named **AWS Cognito** as the user pool. Confirm the layering. | Backend |
| 5 | Token TTL | Governed by `default_access_token_expired_duration` — confirm the concrete value. | Backend |

---

## 17. Test case suite

> Origin: **MOB-2550** · Type: **acceptance** · Status: **active** unless noted.
> **Auto** = automated (yes/no). Full suite = 50 cases.

### 17.1 Happy-path login

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20097 | high | yes | Login with valid **email** (`driver@jitsu.com`) + password | Email shown as typed; OAuth2 triggered; login succeeds → Contract session screen |
| ENG-TC-20098 | high | yes | Login with valid **username** (`john_driver`) + password | Username shown as typed; login succeeds → Contract session screen |
| ENG-TC-20099 | high | yes | Login with valid **phone** (`0912345678`) + password | Phone auto-formatted `(xxx) xxx-xxxx`; login succeeds → Contract session screen |

**Preconditions:** Status=BACKGROUND_APPROVED, Background=MANUAL_APPROVED, Is Linehaul driver=Yes; device online.

### 17.2 Account-type gating

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20100 | high | yes | Valid **DSP** credentials | Rejected — *"Invalid account. Please sign in using your corresponding driver app."* |
| ENG-TC-20101 | high | yes | Valid **PLATFORM** credentials | Rejected — same message |
| ENG-TC-20102 | high | yes | Valid **IC** credentials | Rejected — same message |

### 17.3 Field validation

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20103 | medium | yes | Empty username/email + Login | Inline: `Email or username required` |
| ENG-TC-20104 | medium | yes | Empty phone + Login | Inline: `Phone Required` |
| ENG-TC-20105 | medium | yes | Type >10 digits into phone (`01234567890`) | Input stops after 10 digits; extra digits ignored |
| ENG-TC-20106 | medium | yes | Phone not matching `(xxx) xxx-xxxx` (`123456`) | Inline: `Invalid Phone Number` |
| ENG-TC-20107 | medium | yes | Empty password + Login | Inline: `Password Required` |
| ENG-TC-20108 | low | yes | Username with letters/spaces/hyphens/apostrophes (`John O'Brien`, `user@123`) | No format error; login proceeds |
| ENG-TC-20143 | medium | yes | Whitespace-only username, then valid username + password | Whitespace trimmed; login succeeds → Contract session screen |

### 17.4 Password & placeholders (UI)

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20109 | low | yes | Type password → tap reveal icon → tap again | Masked `•••` → plain text → masked again |
| ENG-TC-20110 | low | yes | View Login screen without tapping | Username/Email placeholder: `Username/Email` |
| ENG-TC-20111 | low | yes | View Login screen without tapping | Password placeholder: `Password` |

### 17.5 Navigation & links

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20112 | low | yes | Look for Sign up link | Sign up NOT present / NOT functional |
| ENG-TC-20113 | medium | yes | Tap **Forgot password** | Forgot Password screen displayed |
| ENG-TC-20114 | medium | yes | Tap **Login with phone number** | Switches to phone method; phone field displayed |
| ENG-TC-20115 | low | yes | Tap **Clear cache** | Delete App pop-up displayed |

### 17.6 Offline & connectivity

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20119 | medium | yes | Open app with no network | Offline banner: yellow bg · white text `Offline - Not connected` · ↑ icon |
| ENG-TC-20120 | high | yes | Login attempt while offline | Blocked; Auth Error pop-up (`Unknown error`) → OK → Login screen |

### 17.7 Token & session

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20121 | high | yes | Reopen app with valid stored token | Login skipped → Contract session screen; no error |
| ENG-TC-20122 | high | yes | Open app with expired/invalid token | Auth Error pop-up (`Unknown error`) → OK → Login screen |
| ENG-TC-20123 | high | **no** | Log in on Device B with Driver A creds | Device B succeeds; Device A force-logged-out → Auth Error pop-up → OK → Login screen |
| ENG-TC-20124 | high | yes | Token expires mid-session, then act | Auth Error pop-up (`Unknown error`) → OK → Login screen |
| ENG-TC-20125 | high | yes | Wrong password N times within window | Account locked; pop-up `Account is locked due to many failed attempts. Retry after <N> mins` |

### 17.8 Account status

| AC ref | Pri | Auto | Precondition (Status / Background / In DSP) | Expected |
|---|---|---|---|---|
| ENG-TC-20127 | medium | yes | CREATED / PENDING / Yes | Login succeeds → 3-session screen |
| ENG-TC-20128 | high | yes | BACKGROUND_APPROVED / MANUAL_APPROVED / Yes (contract signed) | Login succeeds → Active assignment screen |
| ENG-TC-20129 | medium | yes | QUIT / MANUAL_APPROVED / Yes (contract signed) | Login succeeds → Active screen |
| ENG-TC-20130 | high | yes | QUIT / MANUAL_APPROVED / **No** (deleted) | Blocked — *"Invalid account…"* → OK → Login screen |
| ENG-TC-20131 | medium | yes | SUSPENDED / MANUAL_APPROVED / Yes | Login succeeds → Contract session screen |
| ENG-TC-20132 | high | yes | CREATED / NULL / No (IC re-activated) | Blocked — *"Invalid account…"* → OK → Login screen |

### 17.9 Invalid credentials

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20133 | high | yes | Invalid username + invalid password | Rejected — `Invalid login credentials` |
| ENG-TC-20134 | high | yes | Valid username + wrong password | Rejected — `Invalid login credentials` |
| ENG-TC-20135 | high | yes | Valid email + wrong password | Rejected — `Invalid login credentials` |

### 17.10 Login button & keyboard

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20140 | high | yes | Rapid double/spam-tap Login | Only 1 auth request; button disabled/debounced; no duplicate calls |
| ENG-TC-20141 | low | yes | Press & hold Login | Pressed state shown; on release request fires; loading state until response |
| ENG-TC-20142 | medium | yes | Tap phone number field | Numeric keyboard displayed (not alphabetic/full) |

### 17.11 Input, device & localization

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20144 | medium | yes | Paste email/username + password from clipboard | Pasted values accepted; login succeeds → Contract session screen |
| ENG-TC-20145 | low | **no** | Rotate device to landscape with partial input | Layout adjusts; fields/buttons/links visible; input preserved; no overlap/clipping |
| ENG-TC-20146 | low | **no** | Use password-manager autofill | Fields auto-filled; login succeeds → active screen |
| ENG-TC-20156 | low | yes | Device language = Spanish | All UI text + validation errors remain English |

### 17.12 End-to-end full flows

| AC ref | Pri | Auto | Scenario | Expected |
|---|---|---|---|---|
| ENG-TC-20136 | high | yes | E2E email/username login (contract signed) | OAuth2 → backend validates DSP type = Linehaul → success → Active assignment screen |
| ENG-TC-20137 | high | yes | E2E phone login `(xxx) xxx-xxxx` (contract signed) | OAuth2 → backend validates Linehaul → success → Active assignment screen |
| ENG-TC-20138 | high | yes | E2E DSP account | OAuth2 → backend validates DSP ≠ Linehaul → Auth Error pop-up `Invalid login credentials` → OK → Login screen |
| ENG-TC-20139 | high | yes | E2E token auto-login (contract signed) | Reopen → stored token read → validated → Login skipped → active screen |
