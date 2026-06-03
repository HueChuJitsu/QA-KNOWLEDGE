# Function: Log-on (Login)

> **App:** Linehaul Driver App · **Status:** Draft (some validation rules pending PO confirmation)
> **Shared context:** see [../README.md](../README.md) — Architecture (§2), DSP Lifecycle (§3),
> Account Creation Flows (§4).

## 1. Description

Login screen of the Linehaul Driver App. Drivers are **provisioned** (no self sign-up).
Supports two driver populations: **DSP Linehaul** and **3P (third-party)**.

## 2. Authentication

- **Auth provider:** AWS Cognito
- **Identifier types accepted:** username, phone number, email
- **No 2FA** required (including new-device login)
- **No offline login** mode

## 3. Validation rules

### Username
- Required, must not be empty
- Format pattern: `^[a-zA-Z\s'-]+$` (letters, spaces, hyphens, apostrophes)
- Maximum length configurable via remote config

> 🟣 **TBD:** The current pattern does not allow digits or `@`, which conflicts with
> login by phone/email. PO must confirm the field design. (Open Question #3 in README)

### Password
- Required, must not be empty
- **No complexity rules at login** (complexity is enforced only in the Reset Password
  flow — defined in a separate ticket)

## 4. Login eligibility

To log in successfully, an account must:

1. Have `account_type = LINEHAUL` (DSP / Platform / IC drivers are blocked)
2. Have `account_status = enabled` in Cognito
3. Match a valid DSP Driver state — see the state machine in [README §3](../README.md#3-dsp-driver-lifecycle--states)

> **Principle:** Only the **deleted** state (`QUIT` + `In DSP? = No`) blocks login.
> All other states allow login; restricting operational actions is handled by app-side
> gating. **Login ≠ Active.**

## 5. Authentication Error pop-up

A **single unified message** is used for all auth failures, to avoid leaking information:

> *"Your session expired or someone used your account to log in. Please log in again."*

## 6. Session management

- **Single session policy** — login from another device forces logout on the current device
- **Token TTL** — 7 days (TBD — pending Backend confirmation, Open Question #4)
- **No offline tokens** — re-authentication required when the token is invalidated

## 7. Supported hyperlinks

| Link | Purpose |
|---|---|
| Forgot password | Trigger password reset flow (separate ticket) |
| Login with phone number | Switch to phone-based login |
| Clear cache | Wipe app cache (tokens, offline data) |
| Tap logo 7× | Open environment switcher (dev/staging/prod) — QA/dev only |

## 8. Not supported

- ❌ Sign up (drivers are provisioned, no self-registration)

## 9. Account lockout

> 🟣 **TBD** — pending PO + Cognito User Pool configuration. (Open Question #5)

## 10. QA / Test notes

> Fill in happy/edge cases during testing. Suggested cases from the spec:

- ✅ Valid login with username / phone / email (account `LINEHAUL` + `enabled`)
- ❌ `deleted` account (`QUIT` + In DSP No) → cannot login / force logout
- ❌ `account_type` ≠ LINEHAUL (DSP / Platform / IC) → blocked
- Single session: login on device B → device A is force-logged-out, shows pop-up §5
- Empty username / empty password → required-field validation
- Username containing digits or `@` → verify against current pattern (⚠️ conflicts with phone/email — TBD)
- 🐛 **MOB-2606:** crash when logging in with a non-existent gojitsu.com account — verify it is fixed

## 11. Related Jira

- **MOB-2004** — Linehaul Driver App (epic)
- **MOB-2334** — Frontend OAuth (✅ Done)
- **MOB-2543** — Enriched auth spec (reference)
- **MOB-2550** — Backend DSP-type gating (🟡 Blocked)
- **MOB-2606** — Login crash bug (🔴 To Do)
