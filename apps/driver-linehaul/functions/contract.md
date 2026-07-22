# WAT-2055 — TPA (Third-Party Agreement) Signing Guard

> **Ticket:** [WAT-2055](https://gojitsu.atlassian.net/browse/WAT-2055) — integrate TPA API against a real backend
> **App:** Linehaul Driver App
> **Origin:** [MOB-2212](https://gojitsu.atlassian.net/browse/MOB-2212) — delivered TPA guard UI, webview shell, JS message bridge, decline flow, re-sign triggers (against mocked backend)
> **Follow-up:** [MOB-2776](https://gojitsu.atlassian.net/browse/MOB-2776) — backend-integration for the above
> **Reference:** [MOB-2543](https://gojitsu.atlassian.net/browse/MOB-2543) — Enriched Specification

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

## Related Jira

| Ticket | Type | Description |
|---|---|---|
| [WAT-2055](https://gojitsu.atlassian.net/browse/WAT-2055) | Story | Integrate TPA API against real backend (this doc) |
| [MOB-2212](https://gojitsu.atlassian.net/browse/MOB-2212) | Story | TPA guard UI, webview shell, JS bridge, decline flow, re-sign triggers (mocked backend) |
| [MOB-2776](https://gojitsu.atlassian.net/browse/MOB-2776) | Story | Backend-integration follow-up to MOB-2212 |
| [MOB-2543](https://gojitsu.atlassian.net/browse/MOB-2543) | Reference | Enriched Specification |
| [MOB-2607](https://gojitsu.atlassian.net/browse/MOB-2607) | Reference | Test case updates / additional AC groups |
