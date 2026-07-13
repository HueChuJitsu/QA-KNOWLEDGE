# Problem Solve Dashboard (INB-1099)

> PRs: [conductor#34](https://github.com/gojitsucom/conductor/pull/34) · [inbound-webapp#457](https://github.com/gojitsucom/inbound-webapp/pull/457) · [inbound-webapp#458](https://github.com/gojitsucom/inbound-webapp/pull/458)

## 1. Description

The Problem Solve dashboard is the supervisor's view of every shipment in the **no-route sort workflow** for a sortation node. It shows, in one place, each shipment's current state (collected, awaiting routing, routed, re-sorted, deferred, RTS, route-reversed, misplaced) so the supervisor (e.g. Ray) can direct the team and make sure no obligation is lost.

**Actor:** Supervisor (Sort Lead) — see [problem-solve-scenarios](problem-solve-scenarios) Story 6.

**URL (Staging):**

```
https://inbound.staging.gojitsu.com/sortation/{node}/problem-solve?sort={sortKey}
```

| Part | Example | Meaning |
|---|---|---|
| `{node}` | `va-iad-4` | Sortation node / facility code |
| `problem-solve` | — | The Problem Solve dashboard page |
| `?sort={sortKey}` | `?sort=age` | Sort order of the shipment list (e.g. `age` = oldest in workflow first) |

> Example: `https://inbound.staging.gojitsu.com/sortation/va-iad-4/problem-solve?sort=age`

---

## 2. Business Flow

The dashboard does not act on shipments itself — it **observes** the workflow and surfaces state for the supervisor to act on. Each shipment shown is one with no route at induction; the dashboard tracks it until it reaches a terminal state.

```
Shipment enters no-route workflow (no route at induction)
  → appears on Problem Solve dashboard
  → state updates live as workers scan / routing events arrive:
       Collected → Awaiting Routing → Routed (Pending Re-sort) → Re-sorted ✅
  → supervisor uses the board to direct: bring pallet, prioritise routing, defer, RTS
  → shipment leaves the board when it hits a terminal state
```

### Workflow states shown

| State | Meaning |
|---|---|
| **Collected** | In the no-route container, confirmed by scan |
| **Awaiting Routing** | No route yet — waiting on routing/tech support |
| **Routed — Pending Re-sort** | Route assigned, not yet re-sorted to position |
| **Re-sorted** ✅ | Workflow complete — sorted to correct position |
| **Route Reversed** | Previously re-sorted, route removed → re-opened, must be collected back |
| **Misplaced** | Expected scan missing (e.g. fell off pallet) — needs to be located |

### Terminal states

| Terminal State | Trigger | Outcome |
|---|---|---|
| ✅ Re-sorted | Routing done, sorted to correct position | Loaded onto vehicle |
| 🏪 Deferred to storage | Supervisor defers — routing task cancelled, storage task created | Moved to problem-solve area for next shift |
| 📤 RTS | Supervisor decides to return to client | Workflow closed, handoff to RTS process |

---

## 3. Spec / Rules

- The dashboard is **read/observe + direct** — resolution actions (defer, RTS) are initiated by the supervisor, the physical proof comes from worker scans.
- `sort=age` orders the list by time-in-workflow so the oldest/most at-risk shipments surface first.
- A shipment leaves the active board only when it reaches a terminal state (Re-sorted / Deferred / RTS).
- An external **route-reversal** event can re-open a completed (Re-sorted) workflow → the shipment reappears with state **Route Reversed** (see Story 5).
- A **misplaced** flag is raised when expected pallet scans don't all arrive (see Story 7).

---

## 4. Landing Page — Status Panel Config

PS health panel on each sort-center card: **Open** count + **Dwell** (7d median, minutes) + top-5 PS types. Open and Dwell are colored **independently**; card status = worst of the two.

| | |
|---|---|
| **Feature flag** | `enable_problem_solve` — default **OFF**, per region (`RG_<region>` → fallback `AP_DEFAULT`). Flip without deploy; page reload required. |
| **Thresholds** | Open: ≤250 🟢 / ≤500 🟠 / >500 🔴 · Dwell: ≤240 min 🟢 / ≤480 min 🟠 / >480 min 🔴 |
| **Thresholds source** | `src/config/problemSolveThresholds.ts` — hardcoded, **requires code merge to change** |
| **Polling** | Every 5 min; on poll failure keeps last good data, no error shown |
| **"Active" facility** | `activeSession > 0` OR `openTotal > 0` (idle = both zero) |
| **"Critical"** | Facility in a red band on either metric |
| **"Avg Dwell" strip** | Mean of per-facility medians — not a true cross-parcel median |

### Dwell calculation

**Cohort** = parcels open at any point in the last 7 days: `is_terminal = FALSE` OR `(is_terminal = TRUE AND completed_at >= now() - 7d)`. Always ≥ open_total.

**Per-parcel:** `COALESCE(completed_at, now()) - created_at` — elapsed-so-far for open parcels, total time for resolved ones.

**Aggregate:** `percentile_cont(0.5)` per facility in seconds → webapp ÷ 60 → minutes.

**Warehouse 144 example (staging 2026-06-22):** open=5, cohort=12, median=**51.8 min** 🟢
Sorted dwell (min): `0.5 9.7 12 17.2 31.4 [32.4] [71.4] 71.9 74.4 78.8 971.7 985` → median = (32.4+71.4)/2 = 51.8. Two outliers at ~16h would push the average to ~196 min — median absorbs them.

---

# Field Reference

> **⚠️ Important for testing:** Use an account whose scope includes **Warehouse Staff** and **Warehouse Manager** roles only. Accounts with broader scopes (e.g. admin) may bypass permission gates and produce misleading test results.

**URL:** `/sortation/:center_id/problem-solve`
**Scope:** No-route shipments only (V1). All data is facility-scoped via `facilityId` resolved from `center_id`.

---

## Page Header

| Field | Displayed as | Source |
|---|---|---|
| Facility region | `Problem solve · {region}` in the h1 | `sortCenterState.region` (from `sortLayoutAPI.loadCenter`) |
| Time window | Toggle: 24h / 7d / 30d | `timeWindowState` (Recoil atom) — re-fetches summary on change |
| Dwell window | Toggle: 24h / 3d / 7d | `dwellTimeWindowState` (Recoil atom) — re-fetches dwell card only |
| Last updated | Timestamp badge | `summaryState.data.generatedAt` (epoch ms, stamped by server) |
| Refresh | Button | Increments `refreshSignalState` → re-triggers all 3 RPCs |

---

## Section 1 — Open Problem Solves (hero card)

**API:** `GetProblemSolveSummary` → `overview` slice

| Field | Displayed as | Backend field |
|---|---|---|
| Total open | Large number + "parcels" | `s.openTotal` |
| Missing | Breakdown row | `s.missing.count` |
| 7d+ old | Breakdown row | `s.openBreakdown.aged7dPlus` |
| In progress | Breakdown row | `s.openBreakdown.inProgress` |
| Placed | Breakdown row | `s.openBreakdown.placed` |

---

## Section 2 — Completion Rate

**API:** `GetProblemSolveSummary` → `completion` (`c`) slice

| Field | Displayed as | Backend field | Notes |
|---|---|---|---|
| Rate | `{n}%` | `c.ratePct` | Rounded to 2 decimal places |
| Delta vs prior | `▲/▼ {n} pts vs prior 7d` | `c.deltaPtsVsPrior` | Rounded to 2dp; positive = up arrow |
| Sparkline | Line chart | `c.trend` (array of floats) | Last 7 data points |
| Resolved count | Footer: `{n} resolved of {m} created` | `c.resolved` | bigint → number |
| Created count | Footer | `c.created` | bigint → number |

### Formula

Computed on `workflow_instance_views`; the card shows a **flow rate** over the window, not a per-parcel value. Source: `conductor/crates/query/src/problem_solve/{queries.rs → get_problem_solve_completion, model.rs → completion_rate}`.

`ratePct = resolved ÷ created × 100` (0 when `created = 0`). It is a **flow rate**, so `resolved` parcels may have been *opened earlier* — the rate **can exceed 100%** while a backlog drains.

| Count | Window | Per-parcel predicate |
|---|---|---|
| **created** | `[now − W, now]` | `created_at` in window |
| **resolved** | `[now − W, now]` | `current_state = 'completed'` AND `completed_at` in window |
| **prior_created** | `[now − 2W, now − W]` | `created_at` in prior window |
| **prior_resolved** | `[now − 2W, now − W]` | `current_state = 'completed'` AND `completed_at` in prior window |

- `deltaPtsVsPrior` = `rate(W) − rate(prior W)`, in **percentage points**. When the prior window has no created parcels its rate is treated as `0`, so the delta equals the current rate.
- `trend` (sparkline) = per-bucket completion rate, oldest → newest, using the same completed-only predicate per bucket.

> **⚠️ INB-1241 — only `completed` counts, not "terminal".** `resolved` keys off `current_state = 'completed'`, **not** the broader `is_terminal` flag. A workflow can go terminal four ways — `completed`, `failed`, `aborted`, `dismissed` ([conductor `WorkflowInstanceState::is_terminal()`](https://github.com/gojitsucom/conductor/blob/dev/crates/shared/src/enums.rs)) — but only `completed` is a genuine resolution. **Dismissed** (admin retire, INB-1235), **failed**, and **aborted** are excluded from the numerator. Before the fix ([conductor#46](https://github.com/gojitsucom/conductor/pull/46)) the query counted all terminal rows, so dismissing one duplicate inflated the rate (e.g. a bogus 33% on staging). The same completed-only fix also applies to the Dwell, Avg-iterations, and multi-facility 7d-dwell cohorts.
>
> **QA check:** abort/dismiss/fail an open workflow → `resolved` and `ratePct` must **not** increase (Open count drops by 1, but the parcel is not a completion). Only a workflow reaching `completed` raises the rate.

**Warehouse 144 example (staging, 7d window, 2026-06-25):** `created = 23`, `resolved = 12` → `ratePct = 52.17%`. `prior_created = 0` → prior rate = 0 → `deltaPtsVsPrior = +52.17 pts`. Aborting an open workflow left `resolved = 12` unchanged (proving the fix) while Open dropped 11 → 10.

---

## Section 3 — Missing Alert

**API:** `GetProblemSolveSummary` → `overview` slice

| Field | Displayed as | Backend field | Notes |
|---|---|---|---|
| Missing count | Large number | `s.missing.count` | Shipments with no placement >10 min from induct |
| Oldest age | Footer: `Oldest: {duration}` | `s.missing.oldestAgeSeconds` | Formatted by `formatAge()` |

---

## Section 4 — Avg Iterations

**API:** `GetProblemSolveSummary`

| Field | Displayed as | Backend field | Notes |
|---|---|---|---|
| Avg iterations | Ring gauge + `{n} iter` | `s.avgIterations` | Float; displayed to 1dp |
| Target | Subtitle: `target ≈ 1.0` | Hardcoded `1.0` (UI constant) | Backend does not send a target |
| Gauge fill | Arc fill % | Derived: `(avg − 1.0) / (3.0 − 1.0)` | Capped at 3.0 for display |

---

## Section 5 — Dwell Time

**API:** `GetProblemSolveDwell` (independent RPC, own window toggle)

| Field | Displayed as | Backend field | Notes |
|---|---|---|---|
| Induct → Placement median | Primary value | `d.inductToPlacement.medianSeconds` | bigint → number; formatted by `formatDuration()` |
| Induct → Placement avg | Grey secondary | `d.inductToPlacement.avgSeconds` | |
| Placement → Solved median | Primary value | `d.placementToSolved.medianSeconds` | |
| Placement → Solved avg | Grey secondary | `d.placementToSolved.avgSeconds` | |

> Both legs show `—` only when the selected window's cohort is empty. Dwell is **live while a parcel is open** — the tile starts accumulating from when the workflow is created, not when the parcel is placed or resolved.

### Formula

Per-parcel legs are **live while open, frozen once the leg closes**. Source: `conductor/crates/query/src/problem_solve/queries.rs → get_problem_solve_dwell`.

| Leg | Starts | Stops (frozen) | NULL when |
|---|---|---|---|
| **Induct → Placement** | Workflow created at FINAL sort scan (`COALESCE(last_inducted_at, created_at)`) | Placed into a container (`induct_to_placement_seconds` written) | Completed without ever being placed |
| **Placement → Solved** | Placed into a container (`placed_at`) | Workflow resolved (`placement_to_solved_seconds` written) | Never placed |

- `created_at` = timestamp of the **FINAL sort scan** (`evidence.sort_type == "FINAL"`) — the moment the no-route workflow is created (INB-1310). A PRESORT scan with `NO_ROUTE` status does not create a workflow and is not captured.
- `last_inducted_at` = **latest** `ShipmentInductedToSort` timestamp (re-inductions reset it); falls back to `created_at` if not yet received.
- `placed_at` = written when `collect_to_container` task completes (`ShipmentPlacedInContainer`).
- **Cohort:** `(is_terminal = FALSE AND created_at in window) OR (is_terminal = TRUE AND current_state = 'completed' AND completed_at in window)`. Aborted / dismissed / failed excluded — same predicate as INB-1241 ([Section 2](#section-2--completion-rate)).
- **median** = `percentile_cont(0.5)`, **avg** = `AVG`, over non-NULL leg values.
- **Refresh:** the page polls every 5 min automatically; use the Refresh button for an immediate update.

> ⚠️ **Deliberate limitation — open parcels are window-bounded by `created_at`.** A parcel that entered Problem Solve *before* the selected window is excluded even if it's still open and aging. A 7d window measures only recent inflow, not the long-tail stuck backlog.

> **Pending — INB-1268:** Intended to change the dwell start time from FINAL sort scan to the **first** no-route scan (any sort stage), by removing the FINAL gate and anchoring `created_at` to the earliest no-route detection. Currently blocked because INB-1310 re-instated the FINAL gate (merged July 7, 2026). Once resolved, dwell will accumulate from the moment the shipment is first detected as no-route rather than from the final sort confirmation.

**QA checks:**
- [ ] Scan a shipment as no-route at FINAL sort → **Induct → Placement** shows a non-zero, growing value — not `—`.
- [ ] Place that shipment into a no-route container → **Induct → Placement** freezes (value stops growing); **Placement → Solved** starts counting.
- [ ] Resolve the shipment → **Placement → Solved** freezes at the final value.
- [ ] Abort/dismiss the shipment → it must **not** contribute to either dwell leg.

---

## Section 6 — Problem Solve Types

**API:** `GetProblemSolveSummary` → `typeBreakdown[]`

| Field | Displayed as | Backend field | Notes |
|---|---|---|---|
| Type name | Row label | `defToLabel(t.type)` | Looks up `WORKFLOW_DEFS` map; unknown id falls back to raw id |
| Type key | Internal key | `defToProblemType(t.type)` | `no_route_sort` → `no_route`; unknown → `other` |
| Count | Bar + number | `t.openCount` | Bar width is proportional to the max count across all types |

**V1 `WORKFLOW_DEFS` registry** (in `decode.ts`):

| `definitionId` | UI type | UI label |
|---|---|---|
| `no_route_sort` | `no_route` | No route |

---

## Section 7 — Aging Table

**API:** `ListProblemSolveParcels` (own RPC, re-fires on any filter/sort/page change)

### Age band tabs

Counts come from the **summary** (`ageBuckets`), not the parcels RPC.

| Tab label | `band` URL param | `ageBuckets` key |
|---|---|---|
| All | _(omitted)_ | `all` = `s.openTotal` |
| 0 – 1h | `0_1h` | `_0_1h` |
| 1 – 4h | `1_4h` | `_1_4h` |
| 4 – 24h | `4_24h` | `_4_24h` |
| 1 – 3d | `1_3d` | `_1_3d` |
| 3 – 7d | `3_7d` | `_3_7d` |
| 7d+ | `7d_plus` | `_7d_plus` |

### Per-row fields (`AgingShipment`)

| Column | Business logic | Backend proto field | Notes |
|---|---|---|---|
| Shipment ID | | `p.shipmentId` / `p.trackingCode` | Tracking number shown as sub-text; falls back to `''` |
| Client | | `p.clientName` | Falls back to `p.clientId` |
| Age | Time since no-route was detected (`now − created_at`); primary signal for urgency — older parcels need action first | `p.ageSeconds` | Computed server-side; formatted by `formatAge()`; coloured warn at 1d+, crit at 3d+ |
| First scan | When the parcel first entered the warehouse — helps distinguish a new shipment with a routing delay from one that has been sitting unresolved for days | `p.firstScanAt.value` → `p.createdAt.value` | Prefers `firstScanAt`; falls back to `createdAt`; shows `—` if absent |
| Type | The reason the parcel is in problem-solve (e.g. no route) — determines what action the worker needs to take | `defToProblemType(p.workflowDefinitionId)` | Same `WORKFLOW_DEFS` registry as Types Breakdown card |
| Status | Current step in the problem-solve workflow — tells the supervisor whether the parcel still needs collecting, is parked, being worked, missing, or closed | `statusToUi(p.status)` | See status mapping below |
| Location | Physical bin/sort position of the parcel in the warehouse — lets a worker go directly to the parcel without searching | `p.location` | Highlighted differently when text matches `/Last seen/i` (location is stale) |
| Container | The cart/container the parcel is currently sitting in — used to locate the parcel at the container level when no bin address is available | `p.containerId` | Shows `—` if absent |
| Iterations | How many times this parcel has cycled through problem-solve — high iteration count signals a recurring or hard-to-resolve case | `p.iteration \|\| 1` | Defaults to 1 if backend sends 0/null |
| Assigned | Which worker currently owns the active task — shows accountability and helps supervisors redistribute work | `p.assignedTo` | Raw user ID; no name lookup in V1 |

### Status mapping

Status is **derived** by the backend from the workflow's terminal flag, phase, and active tasks — it is not stored directly. Source: `conductor/crates/query/src/status.rs → derive_status()`.

**API:** `ListProblemSolveParcels` — response field `ProblemSolveParcel.status` (proto enum `ProblemSolveStatus`)
**Decode:** `statusToUi(p.status)` in `src/grpc/problemSolve/conductor/decode.ts`

#### No Route workflow

**Derivation rule order** (earlier rule wins):
1. `is_terminal = true` → **Resolved**
2. `locate_package` OR `collection_overdue` task active → **Missing**
3. `routing_obligation` task completed (`routing_resolved = true`) → **In progress**
4. Phase = `no_route_detected` + `collect_to_container` active → **New**
5. Phase = `no_route_detected` + collection done → **Placed**
6. Anything else non-terminal → **In progress**

| Proto enum | Chip label | Logic | Trigger event(s) | Workflow condition |
|---|---|---|---|---|
| `UNSPECIFIED` | New | Safe fallback; treated same as `NEW` | — | — |
| `NEW` | New | No-route detected; not yet collected onto a problem-solve container | `ShipmentNoRouteDetected` (from `SHIPMENT.INBOUND.scan` where `fact.status = "NO_ROUTE"`) **or** `ShipmentUnRouted` (route removed) | Phase = `no_route_detected`, `collect_to_container` task active, routing not yet resolved |
| `PLACED` | Placed | On a problem-solve container, parked — no active task | **Path A:** `STS.SESSION_SHIPMENT_SERVICE.move-shipment-to-container` where `fact.purposes` contains `"no_route"` (container id from `state.container_id`)<br>**Path B:** `STS.SHIPMENT.scan-shipment-for-container` where `fact.purposes` contains `"no_route"` (container id from `ref.uid`, `CNS_` prefix stripped)<br>→ both translate to `ShipmentPlacedInContainer`, completing `collect_to_container` task | Phase = `no_route_detected`, collection done, `routing_obligation` still pending |
| `IN_PROGRESS` | In progress | A re-sort or store task is actively running | **Path A:** `ShipmentRouted` (from `SHIPMENT.PLANNING.add-redelivery` or `SHIPMENT.PLANNING.sprinkle`) — completes `routing_obligation` task<br>**Path B:** Both `collect_to_container` + `routing_obligation` complete → `sort_to_position` spawned, phase advances to `awaiting_re_sort` | `routing_resolved = true`, or phase past `no_route_detected` |
| `MISSING` | Missing | Either overdue marker is active — action required | **Collection overdue:** `ShipmentNoRouteDetected` starts 30-min clock; not placed in container in time → `collection_overdue` spawns.<br>**Re-induction overdue:** `ContainerCommittedToSortInduction` starts 20-min clock; not inducted to sort in time → `locate_package` spawns. See "Missing-detection logic" below. | `locate_package` OR `collection_overdue` task active; overrides all non-terminal states |
| `RESOLVED` | Resolved | Workflow is terminal — problem solve closed | **Path A:** `ShipmentSortedToPosition` — completes `sort_to_position` task, phase = `sorted`, workflow terminal<br>**Path B:** `TriggerManualAction(RTS)` — phase = `rts_exit`, workflow terminal<br>**Path C:** `WorkflowInstanceAborted` — force-aborted via admin RPC | `is_terminal = true` |

> `new` parcels are not broken out as a tile in the Open Problem Solves hero card. Derive the count as: `totalOpen − missing − inProgress − placed`.

#### Missing-detection logic

A parcel becomes **Missing** when it misses a time-bound checkpoint. There are **two timers**:

| Timer | Window | Starts when | Becomes Missing if | Marker task |
|---|---|---|---|---|
| **Collection overdue** | **30 min** | Shipment scanned as no-route (`ShipmentNoRouteDetected`) | Not placed into a no-route container within 30 min | `collection_overdue` |
| **Re-induction overdue** | **20 min** | Container committed to sort induction (`CommitContainerToSortInduction`) | Not inducted to sort within 20 min (anchored to `committed_at`) | `locate_package` |

> **INB-1286:** The `COLLECTION_OVERDUE_MISSING_ENABLED` env flag (default `false`) controls whether `collection_overdue` is counted in the **summary card's Missing count** (SQL). When the flag is off, the summary excludes collection-overdue parcels from the Missing chip. The per-row status (`derive_status()`) **always** treats both markers as Missing regardless of the flag — so a parcel with `collection_overdue` active shows as **Missing** on its row even when the summary count is suppressed.

**Collection overdue (30 min):**
From the moment a shipment is scanned as no-route, a 30-min window opens for a worker to scan it into a no-route container.
- Placed in container in time → `collection_overdue` never spawns.
- Not placed within 30 min → `collection_overdue` spawns → status = **Missing**.

**Re-induction overdue (20 min):**
After the parcel is on a container and that container is committed to sort induction, a 20-min clock starts.
- Inducted in time → clock cancelled (`sort_to_position` spawns), no `locate_package`.
- Not inducted in 20 min → `locate_package` spawns → status = **Missing**.

**QA takeaways:**
- A parcel goes Missing via **either** overdue marker — collection-not-collected (30 min from no-route scan) OR placed-but-never-inducted (20 min from container commit).
- Each window is anchored to the triggering event's timestamp, not to when the system processes it — processing lag does not extend the window.
- Expect a small lag (~seconds) between a deadline and the status flipping, because the countdowns are checked in batches — not a bug.

#### How to resolve a Missing parcel

| Marker | What CLEARS it |
|---|---|
| `collection_overdue` | **Scan into a no-route container** (`ShipmentPlacedInContainer`) — completes `collect_to_container`, cancels `collection_overdue` |
| `locate_package` | **Any induction scan** (`ShipmentInductedToSort`, either `first` or `from_pallet`) — cancels `locate_package` at the top of the induction handler regardless of context |

**Resolution steps for QA:**

| Goal | Action |
|---|---|
| Clear `collection_overdue` | Scan the parcel into a no-route container → **New** or **Placed** |
| Clear `locate_package` | Scan the parcel at **sort induction** (any induction scan) → **In progress** or **Placed** |
| Straight to **Resolved** | Sort the parcel to its final position (`scan-audit` with `state.audited = "true"`, or `put-to-bin`, or `confirm-sort-slot`) → `ShipmentSortedToPosition` |

#### Geocode Failed workflow

A geocode-failed parcel always reads **In progress** for its entire active life — `derive_status()` only special-cases the `no_route_detected` phase for the **New** / **Placed** chips; every other non-terminal phase falls through to rule 6. See [Section 8](#section-8--geocode-failed-workflow-status-flow--resolution) for full resolution paths.

| Phase | Active task(s) | Dashboard status |
|---|---|---|
| `geocode_failed` | `attempt_manual_fix` | In progress |
| `escalated` | `stow_to_geocode_bin` → `investigate_geocode` | In progress |
| `return_to_sender` | `mark_rts` → `stow_to_rts_container` | In progress |
| `resolved` | — (terminal) | **Resolved** |
| `rts_complete` | — (terminal) | **Resolved** (closed via RTS) |

---

## Section 8 — Geocode Failed Workflow (status flow & resolution)

> **Scope note:** the dashboard V1 surfaces **no-route shipments only**. The `geocode_resolution` workflow runs in Conductor in parallel; this section documents its status flow because a geocode-failed parcel that does surface (or is queried directly) always reads as **In progress**, which surprises QA. Source: `conductor/crates/workflows/src/geocode_resolution.rs`.

### What it is

A separate Conductor workflow (`definitionId = geocode_failed`, type `geocode_resolution`) created when a shipment cannot be geocoded — its delivery address is unresolved.

| | |
|---|---|
| **Created by** | `GeocodeFailed` event (creation trigger). Translated from `SHIPMENT.INBOUND.scan` where `fact.status = "GEOCODED_FAILED"`, or from upstream `SHIPMENT.PLANNING.geocode` failure |
| **First task** | `attempt_manual_fix` spawned immediately; phase = `geocode_failed` |

### Why it always shows "In progress" on the dashboard

`derive_status()` only special-cases the `no_route_detected` phase for the **New** / **Placed** chips. Every other non-terminal phase — including `geocode_failed`, `escalated`, `return_to_sender` — falls through to the default **rule 6 → In progress**. So a geocode-failed parcel reads **In progress** for its entire active life until it goes terminal (**Resolved** or RTS). It is never **New** or **Placed**. This is by design today, not a bug — the status model was built for the no-route flow.

### Phase / task flow

| Phase | Active task(s) | Dashboard status |
|---|---|---|
| `geocode_failed` | `attempt_manual_fix` | In progress |
| `escalated` | `stow_to_geocode_bin` → `investigate_geocode` | In progress |
| `return_to_sender` | `mark_rts` → `stow_to_rts_container` | In progress |
| `resolved` | — (terminal) | **Resolved** |
| `rts_complete` | — (terminal) | **Resolved** (closed via RTS) |

### How to resolve a Geocode Failed workflow

Four paths reach a terminal state. The first is automatic; the rest are driven by task completion outcomes.

| # | Path | Trigger | Result |
|---|---|---|---|
| 1 | **Auto re-geocode** (no operator action) | Address gets fixed upstream → legacy re-emits `SHIPMENT.PLANNING.geocode` with `status=GEOCODED` → translated to `ShipmentGeocoded`. The workflow subscribes to this event (`evidence_subscriptions`) and self-completes | phase = `resolved`, **terminal** → **Resolved** |
| 2 | **Manual fix succeeds** | Complete task `attempt_manual_fix` with outcome **`Resolved`** | phase = `resolved`, **terminal** → **Resolved** |
| 3 | **Escalate → investigate → fix** | `attempt_manual_fix` outcome **`Escalate`** → spawns `stow_to_geocode_bin` + `investigate_geocode` (phase `escalated`). Then `investigate_geocode` outcome **`Resolved`** | phase = `resolved`, **terminal** → **Resolved** |
| 4 | **Unresolvable → RTS** | `investigate_geocode` outcome **`Unresolvable`** → spawns `mark_rts` + `stow_to_rts_container` (phase `return_to_sender`). Then `stow_to_rts_container` outcome **`Completed`** | phase = `rts_complete`, **terminal** (return-to-sender, not a successful resolve) |

**Fastest path for QA:** fix the shipment address so upstream re-geocodes → `ShipmentGeocoded` lands on the shipment stream → workflow auto-completes with **no** operator action (Path 1). Alternatively, complete the active `attempt_manual_fix` task with outcome `Resolved` via the Task gRPC / problem-solve UI (Path 2).

> Task completions are recorded as `WorkflowInstanceTaskCompleted` with the `outcome` field (`resolved` / `escalate` / `unresolvable` / `completed`) on the `workflow-instance-{id}` stream. The `outcome` value is what selects the branch above.

---

## Simulating data — calling Conductor gRPC directly

To drive the **Missing (re-induction overdue)** flow without the physical induction step, call `CommitContainerToSortInduction` directly. This commits a no-route container to a sort-induction line and starts the 20-min `locate_package` countdown for every shipment inside it. Conductor uses gRPC-web, so you can call it with `curl`.

### Endpoint

```
POST https://api.conductor.staging.gojitsu.com/conductor.v1.ContainerService/CommitContainerToSortInduction
```

| | |
|---|---|
| **Service / method** | `conductor.v1.ContainerService / CommitContainerToSortInduction` |
| **Content-type** | `application/grpc-web+proto` |
| **Auth** | `Authorization: Bearer <access token>` — grab the `at` cookie value from a logged-in `inbound.staging.gojitsu.com` session (DevTools → Application → Cookies), or copy the `authorization` header from any conductor request in the Network tab |

**Request fields** (proto `CommitContainerToSortInductionRequest`):

| Field | Field # | Required | Notes |
|---|---|---|---|
| `container_id` | 1 | ✅ | The container/cart id (as shown in the Aging table's Container column) |
| `induction_location_id` | 2 | ✅ | Free-form induction line id, e.g. `induction_line_1` |
| `committed_by` | 3 | — | Optional operator id |

**Response** echoes `container_id`, the `shipment_ids` snapshot inside the container, `induction_location_id`, and server `committed_at`.

### Easiest: build the request body with Python

gRPC-web frames the protobuf as `[1 byte flag = 0x00][4-byte big-endian length][protobuf bytes]`. This script builds the body, calls the endpoint, and prints the readable response:

```bash
CONTAINER_ID="fbb7b2f2-dbf8-4b52-ac59-dd04ea4fe687"
INDUCTION_LINE="induction_line_1"
TOKEN="<paste your bearer token>"

python3 -c "
import struct
def es(f,v):
    e=v.encode(); return bytes([(f<<3)|2,len(e)])+e
proto = es(1,'$CONTAINER_ID') + es(2,'$INDUCTION_LINE')
open('/tmp/commit.bin','wb').write(b'\x00'+struct.pack('>I',len(proto))+proto)"

curl -s -D /tmp/hdr.txt \
  'https://api.conductor.staging.gojitsu.com/conductor.v1.ContainerService/CommitContainerToSortInduction' \
  -H 'content-type: application/grpc-web+proto' \
  -H 'x-grpc-web: 1' \
  -H 'origin: https://inbound.staging.gojitsu.com' \
  -H "authorization: Bearer $TOKEN" \
  --data-binary @/tmp/commit.bin -o /tmp/body.bin

grep -i 'grpc-status\|grpc-message' /tmp/hdr.txt
python3 -c "d=open('/tmp/body.bin','rb').read();print(''.join(chr(b) if 32<=b<127 else '.' for b in d))"
```

- **Success** → `grpc-status: 0` and the readable body shows the container id, the `SH_…` ids inside, the line, and the `committed_at` timestamp.
- The 20-min `locate_package` deadline = `committed_at` + 20 min. If those shipments are not inducted to sort by then, they flip to **Missing**.

### Common errors

| `grpc-status` | `grpc-message` | Meaning |
|---|---|---|
| `0` | — | Success |
| `9` (FAILED_PRECONDITION) | `container is already committed to sort induction` | This container was already committed — **commit is one-way, it cannot be undone**. Use a fresh container_id (or have the event-store stream reset by infra). |
| `9` | `container is empty` | No shipments in the container — nothing to commit |
| `16` (UNAUTHENTICATED) | — | Token missing/expired — refresh it from the browser session |

> **You cannot un-commit a container.** Once committed, its status is permanent (Invariant: "one obligation chain, never reopen"). To re-test the commit flow, use a container that has never been committed.

### Verifying the result

- **Workflow reacted** — the shipment's no-route workflow `updated_at` should bump within ~1 s of `committed_at` (the commit event reached the workflow and armed the timer).
- **Missing fired** — after the 20-min deadline a `locate_package` task appears on the workflow and the parcel reads **Missing** on the dashboard (allow a few seconds of batch-polling lag).
- **Cancelled** — if the shipment is inducted to sort before the deadline, `sort_to_position` spawns, the timer is cancelled, and `locate_package` never appears.

---

## QA / Test notes

**Happy path**
- [ ] Open `…/sortation/va-iad-4/problem-solve?sort=age` → board loads with shipments grouped/ordered by their workflow state.
- [ ] Shipment scanned into no-route container → appears as **Collected**.
- [ ] Routing event arrives → state moves to **Routed — Pending Re-sort**, then **Re-sorted** after re-sort scan → drops off the active board.

**Sorting**
- [ ] `?sort=age` → oldest-in-workflow shipments listed first.
- [ ] Remove/change the `sort` param → verify default ordering and that other valid sort keys behave correctly.

**State transitions / edge cases**
- [ ] **Defer** a shipment → routing task cancelled, **Storage** task created, shipment shown as Deferred (not silently dropped).
- [ ] **RTS** a shipment → workflow closed, removed from active board.
- [ ] **Route reversal** of a re-sorted shipment → workflow re-opens, shipment reappears as **Route Reversed** with a collection task.
- [ ] **Misplaced**: pallet consumed expecting N scans but fewer arrive → missing shipment flagged as **Misplaced** with an alert; once re-scanned, flag clears.
- [ ] Abort/dismiss/fail an open workflow → `resolved` and `ratePct` must **not** increase (Open count drops by 1 only).
- [ ] Missing parcel (`collection_overdue`): must be **placed into a no-route container** to clear — re-scanning at conveyor does not clear it.
- [ ] Missing parcel (`locate_package`): **any induction scan** clears Missing.
- [ ] Dwell tile: scan a shipment into Problem Solve at FINAL sort → Induct→Placement dwell shows a non-zero value and grows on refresh (not `—`). Place it → Induct→Placement freezes, Placement→Solved starts live. Abort/dismiss it → it must **not** contribute to either dwell leg.

**Things to watch**
- [ ] Counts on the board match the actual workflow tasks (no double-count when a task transitions).
- [ ] Live update latency — board reflects new scans / routing events without manual refresh.
- [ ] Wrong/unknown `{node}` in the URL → graceful handling (empty/error state, not a crash).
