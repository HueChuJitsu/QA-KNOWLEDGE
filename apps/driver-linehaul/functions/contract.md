# WAT-2055 — TPA (Third-Party Agreement) Signing Guard

> **Ticket:** [WAT-2055](https://gojitsu.atlassian.net/browse/WAT-2055) — integrate TPA API against a real backend
> **App:** Linehaul Driver App
> **Origin:** [MOB-2212](https://gojitsu.atlassian.net/browse/MOB-2212) — delivered TPA guard UI, webview shell, JS message bridge, decline flow, re-sign triggers (against mocked backend)
> **Follow-up:** [MOB-2776](https://gojitsu.atlassian.net/browse/MOB-2776) — backend-integration for the above
> **Reference:** [MOB-2543](https://gojitsu.atlassian.net/browse/MOB-2543) — Enriched Specification
> **Test cases origin:** MOB-2212 · **Last updated by:** [MOB-2607](https://gojitsu.atlassian.net/browse/MOB-2607) (2026-07-14)

---

## 1. Description

A **top-level TPA (Third-Party Agreement) signing guard** that blocks app access until the driver has signed the agreement. The TPA is displayed via a **webview** loading from the legal management system (`dashboard.gojitsu.com/legal-management`) and uses **JavaScript messaging** to signal completion back to the app.

The guard sits **above tab navigation**, similar to the existing auth guard pattern — while a TPA is unsigned, no other part of the app is reachable.

## 2. Scope

### In scope
- Top-level TPA guard (above tab navigation, similar to the auth guard pattern)
- Webview loading TPA content from the legal management system
- JavaScript message channel from webview → Flutter app
- Backend TPA status verification after the JS message is received
- TPA version detection (a new version triggers re-signing)
- Interruption handling: return to the TPA screen on next app open

### Out of scope
- Login / authentication (separate ticket)
- App shell and navigation (separate ticket)
- Profile sections (separate tickets)

## 3. Happy Path Flow

1. Driver logs in (handled by the auth ticket)
2. App checks TPA status — if unsigned (or a new version / annual renewal is due), the TPA guard activates
3. Driver sees the TPA webview, loaded from the legal management system
4. Driver reads and signs the TPA inside the webview
5. Webview sends a JavaScript message to the app signaling completion
6. App verifies the signed status with the backend
7. TPA guard releases; driver proceeds to the app shell

> If a driver has **more than one pending TPA**, they are signed **in sequence** — each signed TPA reveals
> the next webview, and navigation stays blocked until all pending TPAs are signed.

## 4. Re-signing triggers

| Trigger | Behavior |
|---|---|
| TPA version updated (previous version signed) | Guard re-activates on next app open; driver must sign the new version |
| Annual periodic renewal (`expired_ts` = signing date + 10 years) | Push notification sent at `expired_ts − 7 days`; guard activates on login within the renewal window; on completion `expired_ts` is updated to new signing date + 10 years |
| No pending TPA for the driver | Guard does not activate; driver bypasses straight to the app shell |

## 5. Errors & Edge Cases

| Condition | User sees | User can do |
|---|---|---|
| TPA webview fails to load | Error state on TPA screen | Retry loading; contact support if persistent |
| App backgrounded during signing | Returns to TPA screen on next open | Continue signing |
| Network drops during signing | Webview shows error or stalled state | Retry when connectivity is restored |
| Device offline before TPA screen opens (API call to fetch webview URL fails) | Server-driven error message shown instead of webview content, no "Retry" option | — |
| JS message received but backend verification fails | Error message with retry option on TPA screen; webview is **not** reloaded; guard is **not** released | Tap retry → re-attempts backend verification only |
| Region identifier invalid / undetermined | Error state: **"Contact Your DSP"** | — |
| TPA version updated after previous signing | TPA guard re-activates | Sign the new version |
| Annual renewal due | TPA guard re-activates | Sign again |

## 6. Acceptance Scenarios

- Given a Linehaul DSP driver who has never signed the TPA, when they log in, then they see the TPA webview and cannot access the app until they sign
- Given a driver who has signed the current TPA version, when they open the app, then they bypass the TPA guard
- Given a driver in the middle of TPA signing, when the app is backgrounded or network drops, then they return to the TPA screen on next app open
- Given a driver who previously signed TPA v1, when TPA v2 is published, then the driver is prompted to re-sign on next app open
- Given a driver whose TPA is expired, when they open the app, then they are prompted to re-sign

---

## Test Cases

> Origin: MOB-2212 · Last updated by: MOB-2607 · Total: 14 test cases · Last updated: 2026-07-14

### Login Gate

#### [TC-01] Never-signed driver, single pending TPA → TPA screen blocks app access

**AC ref:** MOB-2607/Group2-AC-1 · **Priority:** High · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- Linehaul DSP driver account that has never signed the TPA
- Driver has exactly one pending TPA
- Backend reports TPA status as unsigned
- Device has network connectivity

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Complete login as a Linehaul DSP driver who has never signed the TPA | Valid driver credentials; TPA status: unsigned; one pending TPA | Login succeeds; the app checks TPA status with the backend |
| 2 | Observe the screen immediately after the TPA status check | — | The TPA screen is displayed with the TPA webview loaded |
| 3 | Attempt to navigate to any other app screen or tab | — | Navigation is blocked; the TPA screen remains displayed until the driver signs the TPA |

#### [TC-02] Never-signed driver, multiple pending TPAs → signed in sequence

**AC ref:** MOB-2607/Group2-AC-2 · **Priority:** High · **Type:** Acceptance · **Automated:** No

**Preconditions:**
- A Linehaul DSP driver has never signed TPA
- Driver has more than one pending TPA
- Device has network connectivity

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Log in as a Linehaul DSP driver who has more than one pending TPA | TPA status: unsigned; multiple pending TPAs | Login succeeds; the TPA screen is displayed with the first TPA webview |
| 2 | Sign the first TPA in the webview | — | The next TPA webview is displayed; navigation to other app screens remains blocked |
| 3 | Sign all remaining TPAs in sequence | — | After each TPA is signed, the next TPA webview appears; navigation stays blocked until all are signed |
| 4 | Observe the screen after all TPAs are signed | — | The TPA screen closes; the app shell (tab navigation) is displayed and the driver can access the app |

#### [TC-03] Driver signed current TPA version → bypasses guard

**AC ref:** MOB-2607/Group4-AC-1 · **Priority:** High · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- A driver has signed the current version of TPA
- Backend reports TPA signed status as current
- Device has network connectivity

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Log in as a driver who has signed the current TPA version | TPA status: signed, current version | Login succeeds; the app performs the TPA status check with the backend |
| 2 | Observe the screen after the TPA status check | — | The TPA screen is not shown; driver bypasses the guard and the app shell is displayed directly |

#### [TC-04] Guard sits above tab navigation — tab bar not interactive while unsigned

**AC ref:** MOB-2212/AC-07 · **Priority:** Medium · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- A logged-in driver has an unsigned TPA
- The TPA screen is displayed

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | With an unsigned TPA, observe the navigation state on the TPA screen | — | The TPA screen is displayed above the tab navigation; the tab bar is not interactive |
| 2 | Attempt to navigate to any app tab | — | Navigation is not permitted; the TPA screen remains displayed and no tab content is shown |

#### [TC-05] No TPA available for the driver → bypass signing flow

**AC ref:** MOB-2607/Group3-AC-7 · **Priority:** High · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- A driver opens the app
- The backend returns no available TPA for the driver

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Open the app; the app fetches the TPA list from the backend | Backend TPA response: no available TPA | The backend returns an empty TPA list for the driver |
| 2 | Observe the screen after the TPA list fetch | — | No TPA signing prompt is triggered; driver bypasses the flow and the app shell is displayed normally |

### Signing

#### [TC-06] Sign TPA in webview → JS message → backend verification success

**AC ref:** MOB-2607/Group3-AC-1 · **Priority:** High · **Type:** E2E · **Automated:** Yes

**Preconditions:**
- A Linehaul DSP driver who has never signed the TPA is logged in
- The TPA screen is displayed with the TPA webview loaded
- Device has network connectivity

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Read and sign the TPA inside the webview on the TPA screen | — | The webview sends the JS signing-complete message to the app |
| 2 | Allow the app to verify the signed status with the backend | Backend verification: success | The backend confirms TPA signed status |
| 3 | Observe the screen after successful verification | — | The TPA webview closes; no additional message state is shown; the app shell is displayed and driver can access the app normally |

#### [TC-07] Interrupted signing (background / network drop) → resumes on next open

**AC ref:** MOB-2607/Group3-AC-2 · **Priority:** High · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- A Linehaul DSP driver is in the middle of TPA signing on the TPA screen
- TPA signing has not been completed

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Begin signing the TPA on the TPA screen without completing it | — | TPA signing is in progress and not completed |
| 2a | [Scenario A] Background the app (e.g. press Home) | — | The app moves to the background |
| 2b | [Scenario B] Disable network connectivity during signing | Device in offline / airplane mode | The TPA webview shows an error or stalled state |
| 3 | Reopen the app (restore connectivity if Scenario B) | — | On next app open, the TPA screen is displayed and the driver must complete signing before accessing the app |

#### [TC-08] Device offline before TPA screen opens → server-driven error, no webview

**AC ref:** MOB-2607/Group3-AC-3 · **Priority:** High · **Type:** Acceptance · **Automated:** No

**Preconditions:**
- Driver has a pending TPA to sign
- Device is offline before the TPA signing screen is opened
- The TPA guard triggers and attempts to load the webview via API

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Device goes offline; driver opens the app (or the guard triggers navigation to the signing screen) | Device offline | The TPA signing screen is shown; the app attempts to fetch the webview URL from the API |
| 2 | The API call to retrieve the TPA webview URL fails due to no network | — | The webview cannot be loaded; the server-driven error message from BE is displayed instead of webview content |

#### [TC-09] Webview fails to load (URL unreachable) → "Contact Your DSP", no Retry

**AC ref:** MOB-2607/Group3-AC-4 · **Priority:** Medium · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- A driver has a pending TPA to sign
- The TPA webview fails to load (e.g. legal management URL unreachable)

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Open the TPA screen; the app attempts to load the TPA webview | TPA webview URL unreachable | The TPA webview fails to load |
| 2 | Observe the TPA screen after the load failure | — | "Contact Your DSP" text is displayed and a "Logout" button is visible; no "Retry" option is present |

#### [TC-10] JS message received but backend verification fails → retry without reload

**AC ref:** MOB-2607/Group4-AC-2 · **Priority:** High · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- A driver is on the TPA webview screen and has just completed signing
- The JS signing-complete message has been sent to the app
- The backend TPA status verification call will return a failure

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Sign the TPA in the webview so the app receives the JS signing-complete message | — | The app receives the JS signing-complete message from the webview |
| 2 | Allow the app to call backend TPA status verification; backend returns a failure | Backend verification: failure | An error message with a retry option is displayed; the webview is **not** reloaded; the guard is **not** released |
| 3 | Tap the retry option on the TPA screen | — | The app retries backend TPA status verification without reloading the webview |

#### [TC-11] Fallback text (webview load failure) respects device locale

**AC ref:** MOB-2607/Group3-AC-5 · **Priority:** Low · **Type:** Acceptance · **Automated:** No

**Preconditions:**
- A driver has a pending TPA to sign
- Device locale is set to the target language (English or Spanish)

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 (Scenario 1 — webview loads) | Open the TPA screen with a working network; wait for the webview to load fully | Device locale: English / Spanish | The TPA document renders inside the webview; no app-owned UI strings (e.g. "Logout", "Contact Your DSP") are visible; i18n check does not apply to webview content |
| 1 (Scenario 2 — webview fails) | Open the TPA screen while the webview cannot be loaded; observe the fallback state | English → "Logout" / "Contact Your DSP" · Spanish → "Cerrar sesión" / "Contacta a tu DSP" | The "Logout" button and "Contact Your DSP" text display in the driver's device locale; no Retry option present |

#### [TC-12] Invalid / undetermined region → "Contact Your DSP" error

**AC ref:** MOB-2607/Group3-AC-6 · **Priority:** High · **Type:** Acceptance · **Automated:** Yes

**Preconditions:**
- A driver has a pending TPA to sign
- The app cannot determine the driver's region identifier, or the identifier is invalid

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Open the app; the app attempts to fetch the TPA URL for the driver's region | Region identifier: invalid or undetermined | The TPA URL fetch fails due to invalid or undetermined region |
| 2 | Observe the TPA screen | — | An error state is displayed with the message "Contact Your DSP" |

### Re-sign

#### [TC-13] New TPA version published (status "NOT YET AGREED") → re-sign guard activates

**AC ref:** MOB-2607/Group5-AC-1 · **Priority:** High · **Type:** Acceptance · **Automated:** No

**Preconditions:**
- A driver has previously signed TPA v1
- A new TPA version is published in the backend with status "NOT YET AGREED"
- Driver is on the login screen

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Log in as a driver who previously signed TPA v1 | TPA v1: signed; new TPA version: status = NOT YET AGREED | Login succeeds; the app checks TPA status with the backend |
| 2 | Observe the screen after the TPA status check | — | The guard re-activates; the TPA screen is displayed with the new TPA webview; app access is blocked until the driver signs the new TPA |

#### [TC-14] Annual renewal window (`expired_ts − 7 days`) → push notification + re-sign guard

**AC ref:** MOB-2607/Group5-AC-2 · **Priority:** High · **Type:** Acceptance · **Automated:** No

**Preconditions:**
- Driver previously signed the TPA; `expired_ts` = signing date + 10 years
- Current date is within 7 days before `expired_ts` (i.e. `now >= expired_ts − 7 days`)
- Push notifications are enabled for the app

| Step | Action | Data | Expected |
|---|---|---|---|
| 1 | Wait for or simulate `now = expired_ts − 7 days` | `expired_ts − 7 days` reached | A push notification is delivered informing the driver it is time to re-sign the TPA contract |
| 2 | Driver logs into the app any time on or after `expired_ts − 7 days` | Login attempt within the 7-day renewal window | The TPA signing guard is displayed immediately after login; driver cannot access any other part of the app until re-signed |
| 3 | Driver completes the TPA re-signing flow | — | The guard is dismissed; `expired_ts` is updated to the new signing date + 10 years; driver can access the app normally |

---

## Related Jira

| Ticket | Type | Description |
|---|---|---|
| [WAT-2055](https://gojitsu.atlassian.net/browse/WAT-2055) | Story | Integrate TPA API against real backend (this doc) |
| [MOB-2212](https://gojitsu.atlassian.net/browse/MOB-2212) | Story | TPA guard UI, webview shell, JS bridge, decline flow, re-sign triggers (mocked backend) |
| [MOB-2776](https://gojitsu.atlassian.net/browse/MOB-2776) | Story | Backend-integration follow-up to MOB-2212 |
| [MOB-2543](https://gojitsu.atlassian.net/browse/MOB-2543) | Reference | Enriched Specification |
| [MOB-2607](https://gojitsu.atlassian.net/browse/MOB-2607) | Reference | Test case updates / additional AC groups |
