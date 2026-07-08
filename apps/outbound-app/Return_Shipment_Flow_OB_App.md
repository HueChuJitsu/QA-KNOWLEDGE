# Return Shipment Flow – Outbound App

> This document is centered on the **return flow in the Outbound App** as a whole. MOB-2926 is only used as a supplementary source — it's the one ticket that documents the "blocked reason / notice" logic, which isn't covered anywhere else in Jira/Confluence.

## 1. Return actions in the Outbound App

When a clerk scans a shipment in the Outbound App, one or more return-related actions may appear depending on the shipment's state. High level:

| Action | Result |
| --- | --- |
| **Mark as Returned** | Shipment moves to a RETURN state (returned to warehouse as usual) |
| **Ready to Return to Client** | Clerk enters a reason — Confirm — status becomes `RETURNED_TO_CLIENT_PENDING` |
| **Mark as Returned to Client** | Status becomes `RETURNED_TO_CLIENT_SUCCEEDED` |
| **Returned, Not Attempted** | Status → `RETURN_SUCCEEDED`, dropoff status → `FAILED`, a new return record is added with status `SUCCEEDED` |

The exact gating logic below was verified directly against `axlehire-data-orchestrator` (`dev` branch, post MOB-2926 merge) rather than taken from docs alone — some conditions here aren't mentioned in any Jira/Confluence page.

**Mark as Returned** — `ShipmentOutboundActionBuilder.buildReturnAction()`
- Shipment's inbound status is `null` or one of `RECEIVED_OK` / `RECEIVED_DAMAGED` / `RECEIVED`
- Not currently mid inbound-processing (`!processInbound`)
- Shipment is on an assignment with stop data loaded (`assignmentId != null`, stops present)
- Shipment has not already been returned to client
- Shipment's own dropoff stop must be in a final state (Succeeded/Failed/Discarded) — otherwise the action is hidden and a "not final" notice is shown instead (see §2)
- No other DROP_OFF/PICK_UP stop on the same route may still be incomplete, and the shipment's return-stop history must allow another return (`ShipmentOutboundManager.canReturnShipment()`) — otherwise the action is hidden and a "Return blocked" notice (with the blocking-stop count) is shown instead (see §2)

**Ready to Return to Client** — `ShipmentOutboundActionBuilder.buildReturnToClientAction()`, first branch
- Shipment has not already been returned to client
- `ShipmentOutboundManager.isAbleReturnToClient()` is true: shipment status is in the eligible-status Consul list AND the client is enabled (in the client list, or return-to-all, or small-parcel-enabled)
- No other route stop may still be incomplete
- If the route isn't done, or the status/client isn't eligible, the action is hidden and the corresponding reason is appended to the notice instead (see §2)

**Mark as Returned to Client** — same builder, second branch
- Not mid inbound-processing
- Shipment status is currently `RETURNED_TO_CLIENT_PENDING`
- Shipment status is not already `RETURNED_TO_CLIENT_SUCCEEDED`
- Pure status-transition gate — no notice logic is involved here

**Returned, Not Attempted** — `ShipmentOutboundActionBuilder.buildReturnNotAttemptedAction()` — an independent condition set from the three actions above
- Shipment is on an assignment
- The assignment's region must be enabled for this feature via Consul (region list contains `*` or the specific region code) — **this region gate isn't documented anywhere in Confluence**, only found in code
- Pickup stop succeeded
- None of the shipment's dropoff stops is already marked "returned, not attempted" (prevents re-triggering)
- None of the shipment's dropoff stops has reached a final state yet (dropoff still pending — matches the Confluence description)
- No other pickup/dropoff stop on the same assignment is still in progress
- No notice logic is involved here either

Notes:
- A clerk can **rescan** a shipment already at `RETURNED_TO_CLIENT_SUCCEEDED` to flip it back to `RETURNED_TO_CLIENT_PENDING` or to `Mark as Returned` — the flow isn't a one-way gate.
- Event history for these actions is visible in the Dispatch App.
- Sources: [Return a shipment to a client](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/579862534/Return+a+shipment+to+a+client), [Returned, Not Attempted](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1301151756/Option+to+Mark+a+Package+as+Returned+Not+Attempted+in+the+Outbound+App), and direct code review (`ShipmentOutboundManager.java`, `ShipmentOutboundActionBuilder.java`, `ShipmentOutboundManagerTest.java`).

## 2. What gates each action — and what the clerk sees when it's blocked

Four independent conditions decide whether **Mark as Returned** and **Ready to Return to Client** are offered. All of this is evaluated server-side, in **axlehire-data-orchestrator (dataorch)**, class `ShipmentOutboundManager`, every time the app calls `/outbound/v3/outbound-info` on scan — the app itself doesn't compute eligibility, it only renders what the backend returns.

Before MOB-2926, a failed condition simply **hid** the corresponding action — the clerk had no way to know why, which led to tech-support tickets or manual "unroutes" (losing the shipment's history). MOB-2926 closed that gap: each failed condition now also produces a **reason**, appended into `ShipmentOutboundInfo.notice`. The app already rendered `notice` as a red banner on the shipment-info screen, so no app-side change was needed — the reason simply shows up proactively on scan, before the clerk taps anything.

| # | Condition | Data checked | Action(s) blocked | Reason shown to clerk (since MOB-2926) |
| --- | --- | --- | --- | --- |
| 1 | **Is the route done?** | Other DROP_OFF/PICK_UP stops on the same route — is any still incomplete (e.g. EN_ROUTE)? | Both **Mark as Returned** and **Ready to Return to Client** | "Return blocked" — includes the count of blocking stops |
| 2 | **Is the shipment's own delivery stop final?** | The shipment's dropoff status (must be Succeeded/Failed/Discarded) | **Mark as Returned** | "Return not available" (not final) |
| 3 | **Is the shipment status eligible for return-to-client?** | Whether `shipment.status` is in the `shipment_statuses_can_be_returned_to_client` list (Consul) | **Ready to Return to Client** | "Return to Client not available" — includes the current status |
| 4 | **Is the client enabled to receive returns?** | Client is in `clients_who_want_to_receive_return_shipments`, OR `enable_return_to_client_to_all = true`, OR shipment is small-parcel and `enable_return_to_client_for_small_parcel = true` | **Ready to Return to Client** | "Return to Client not available" — client not enabled |

The notice mechanism is scoped to **only these two actions**. It's implemented directly inside `buildReturnAction()` and `buildReturnToClientAction()` via shared helpers (`appendNotice` / `appendReturnBlockedNotice`), not as a separate pass — and confirmed absent from **Mark as Returned to Client** and **Returned, Not Attempted**, which never touch `ShipmentOutboundInfo.notice`.

Additional rules on top of the table above:
- **Dedup** — the helper checks whether the message text is already a substring of the existing notice before appending; since condition #1 blocks both Return and Return to Client at once, this means only a single "Return blocked" reason ends up shown, not repeated.
- **Append, not overwrite** — a new reason is appended to any existing notice with a blank-line separator, rather than replacing it.
- A shipment already at `RETURN_SUCCEEDED` / `RETURNED_TO_CLIENT_SUCCEEDED` produces no spurious reason.
- Message content is configurable via Consul (see §3); each key also has a hardcoded default string in `ShipmentOutboundManager` used when Consul has no value. If a message's placeholder is malformed, `String.format` fails safely and the raw (unformatted) template is shown instead of crashing.
- Out of scope: a shipment previously marked MISSING after `RECEIVED_OK` (needs to be rescanned before it can be added to a route again) — not covered by this logic.

This is entirely a backend change (dataorch only, no Outbound App release needed), and it's backed by `ShipmentOutboundManagerTest` — 127 unit tests covering all four conditions, small-parcel/return-to-all eligibility, Consul overrides, the bad-placeholder fallback, dedup, and append-to-existing-notice (full module: 466 tests passing).

Source: [MOB-2926](https://gojitsu.atlassian.net/browse/MOB-2926) (description + Thuan Nguyen's implementation comment + Trang Le's staging/beta/prod verification comments).

## 3. Where it's configured (Consul)

**Eligibility conditions** (rows #3–#4 above) — path `apps.dataorch.attributes.<key>`:

| Key | Meaning |
| --- | --- |
| `clients_who_want_to_receive_return_shipments` | Comma-separated list of client IDs enabled to receive returns |
| `enable_return_to_client_to_all` | true/false — enables it for all clients |
| `enable_return_to_client_for_small_parcel` | true/false — enables it specifically for small-parcel shipments |
| `shipment_statuses_can_be_returned_to_client` | List of statuses eligible for return-to-client (on prod: GEOCODED, GEOCODE_FAILED, RETURN_SUCCEEDED, PICKUP_FAILED, DROPOFF_FAILED, CANCELLED_BEFORE_PICKUP, CANCELLED_AFTER_PICKUP, DISPOSABLE, INVALID_ADDRESS, HANDED_OFF, UNSERVICEABLE, UNDELIVERABLE, DAMAGED, RETURN_DAMAGED, RESCHEDULED) |

**Notice message content** (from MOB-2926) — path `apps.dataorch.outbound.<key>`; falls back to a built-in default if unset:

| Key | Placeholder |
| --- | --- |
| `outbound_return_blocked_message` | `%d` = number of blocking stops |
| `outbound_return_not_final_message` | — |
| `outbound_return_to_client_status_not_eligible_message` | `%s` = current status |
| `outbound_return_to_client_client_not_enabled_message` | — |

**A dataorch restart is required after changing any of the keys above** for the new value to take effect. The built-in default values (used when a Consul key is unset) live in code, in the `attributes` map of `config.yaml` and `config-staging-local.yaml` in dataorch.

Sources: [Configure to allow a client to receive a returned shipment](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/579829761/Configure+to+allow+a+client+to+receive+a+returned+shipment), Trang Le's prod-verification comment and Thuan Nguyen's implementation comment on MOB-2926.

## 4. References

- [Return a shipment to a client](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/579862534/Return+a+shipment+to+a+client)
- [Configure to allow a client to receive a returned shipment](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/579829761/Configure+to+allow+a+client+to+receive+a+returned+shipment)
- [Option to Mark a Package as "Returned, Not Attempted"](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/1301151756/Option+to+Mark+a+Package+as+Returned+Not+Attempted+in+the+Outbound+App)
- [OUTBOUND APP FLOW](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/562135067/OUTBOUND+APP+FLOW) (general state-based flow table — dated 2022, may not fully reflect the current app)
- [MOB-2926](https://gojitsu.atlassian.net/browse/MOB-2926) — source for §2's reason/notice logic and the prod Consul values in §3
- Code: `ShipmentOutboundManager.java`, `ShipmentOutboundActionBuilder.java`, `ShipmentOutboundManagerTest.java` in the `axlehire-data-orchestrator` repo (`dev` branch)
