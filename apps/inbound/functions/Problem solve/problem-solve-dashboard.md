# Landing Page — Problem Solve Status Panel (INB-1099)

> PRs: [conductor#34](https://github.com/gojitsucom/conductor/pull/34) · [inbound-webapp#457](https://github.com/gojitsucom/inbound-webapp/pull/457) · [inbound-webapp#458](https://github.com/gojitsucom/inbound-webapp/pull/458) ⚠️ open

PS health panel on each sort-center card: **Open** count + **Dwell** (7d median, minutes) + top-5 PS types. Open and Dwell are colored **independently**; card status = worst of the two.

## Config

| | |
|---|---|
| **Feature flag** | `enable_problem_solve` — default **OFF**, per region (`RG_<region>` → fallback `AP_DEFAULT`). Flip without deploy; page reload required. |
| **Thresholds** | Open: ≤250 🟢 / ≤500 🟠 / >500 🔴 · Dwell: ≤240 min 🟢 / ≤480 min 🟠 / >480 min 🔴 |
| **Thresholds source** | `src/config/problemSolveThresholds.ts` — hardcoded, **requires code merge to change** |
| **Polling** | Every 5 min; on poll failure keeps last good data, no error shown |
| **"Active" facility** | `activeSession > 0` OR `openTotal > 0` (idle = both zero) |
| **"Critical"** | Facility in a red band on either metric |
| **"Avg Dwell" strip** | Mean of per-facility medians — not a true cross-parcel median |

## Dwell calculation

**Cohort** = parcels open at any point in the last 7 days: `is_terminal = FALSE` OR `(is_terminal = TRUE AND completed_at >= now() - 7d)`. Always ≥ open_total.

**Per-parcel:** `COALESCE(completed_at, now()) - created_at` — elapsed-so-far for open parcels, total time for resolved ones.

**Aggregate:** `percentile_cont(0.5)` per facility in seconds → webapp ÷ 60 → minutes.

**Warehouse 144 example (staging 2026-06-22):** open=5, cohort=12, median=**51.8 min** 🟢
Sorted dwell (min): `0.5 9.7 12 17.2 31.4 [32.4] [71.4] 71.9 74.4 78.8 971.7 985` → median = (32.4+71.4)/2 = 51.8. Two outliers at ~16h would push the average to ~196 min — median absorbs them.

---

# Problem Solve Dashboard — Field Reference

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

> Both legs show `—` when the selected window has no resolved parcels (all fields return 0).

### Formula

Per-parcel legs stored on `workflow_instance_views`; the card shows median/avg **across all resolved parcels in the window** (not one shipment). Source: `conductor/crates/query/src/{workflow_instance/projector.rs, problem_solve/queries.rs}`.

| Leg | Per-parcel formula |
|---|---|
| **Induct → Placement** | `placed_at − last_inducted_at` |
| **Placement → Solved** | `completed_at − placed_at` |

- `last_inducted_at` = **latest** induct (re-inductions reset it). `placed_at` = `collect_to_container` completes. `completed_at` = workflow terminal (e.g. `ShipmentSortedToPosition`).
- **Cohort** = `is_terminal = TRUE` AND `completed_at` within window (+ facility/type filter). In-progress parcels are excluded from both legs. `Placement → Solved` is `NULL` if the parcel resolved without a placement.
- **median** = `percentile_cont(0.5)`, **avg** = `AVG`, over non-NULL values. With 2 values they're equal (midpoint); diverge at ≥3.

**Example** — cohort `[SH_70439296, SH_70439294]`: Induct→Placement `[5m41, 7m31]` → median/avg **6m36**; Placement→Solved `[7m37, 19m34]` → median/avg **13m36**. (With only `SH_70439296` resolved the card read 5m41 / 7m37 — the value shifts as parcels resolve.)

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
2. `locate_package` task active → **Missing**
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
| `MISSING` | Missing | `collection_overdue` **or** `locate_package` task is live — a checkpoint timer expired, action required | **Reason A — collection overdue (30 min):** no-route detected → 30-min countdown starts; parcel NOT placed onto a problem-solve container in time → `collection_overdue` spawns. Cancelled if the parcel is placed first.<br>**Reason B — re-induction overdue (20 min):** `ContainerCommittedToSortInduction` arrives → 20-min countdown anchored to `committed_at`; parcel NOT inducted to sort in time → `locate_package` spawns. Cancelled if `ShipmentInductedToSort` arrives first.<br>See "Missing-detection logic" below for the full two-timer model. | `collection_overdue` or `locate_package` task active; overrides all non-terminal states |
| `RESOLVED` | Resolved | Workflow is terminal — problem solve closed | **Path A:** `ShipmentSortedToPosition` — completes `sort_to_position` task, phase = `sorted`, workflow terminal<br>**Path B:** `TriggerManualAction(RTS)` — phase = `rts_exit`, workflow terminal<br>**Path C:** `WorkflowInstanceAborted` — force-aborted via admin RPC | `is_terminal = true` |

> `new` parcels are not broken out as a tile in the Open Problem Solves hero card. Derive the count as: `totalOpen − missing − inProgress − placed`.

#### Missing-detection logic

A parcel becomes **Missing** when it fails to reach an expected checkpoint within a time window. There are **two independent timers**, each guarding a different stage of the no-route flow. Both flip the parcel to **Missing**, but for different reasons (surfaced in the Missing dialog). The workflow cancels one before arming the other, so in practice only one is active at a time.

| Timer | Window | Starts when | Becomes Missing if | Marker task |
|---|---|---|---|---|
| **Collection overdue** | **30 min** | No-route is detected (workflow created) | Parcel is **not placed onto a problem-solve container** within 30 min | `collection_overdue` |
| **Re-induction overdue** | **20 min** | Container holding the parcel is committed to sort induction (`CommitContainerToSortInduction`) | Parcel is **not inducted to sort** within 20 min (anchored to the event's `committed_at`) | `locate_package` |

**Stage 1 — Collection overdue (30 min):**
When a parcel is detected as no-route, a 30-min clock starts. The parcel is expected to be collected/placed onto a problem-solve container within that time.
- Placed in time → clock cancelled, no `collection_overdue`, status goes **Placed**.
- Not placed in 30 min → `collection_overdue` spawns → status = **Missing** (reason: *not collected within the window*).

**Stage 2 — Re-induction overdue (20 min):**
After the parcel is on a container and that container is committed to sort induction, a 20-min clock starts. The parcel is expected to show up at sort induction within that time.
- Inducted in time → clock cancelled (`sort_to_position` spawns), no `locate_package`.
- Not inducted in 20 min → `locate_package` spawns → status = **Missing** (reason: *not re-inducted after commit*).

**QA takeaways:**
- Two ways a parcel goes Missing: **never placed** (30 min after no-route detected) or **placed but never inducted** (20 min after the container is committed).
- Committing a container does **not** by itself make a parcel Missing — it only starts the 20-min clock.
- Each window is anchored to the triggering event's timestamp, not to when the system processes it — processing lag does not extend the window.
- Expect a small lag (~seconds) between a deadline and the status flipping, because the countdowns are checked in batches — not a bug.
- On re-induction the collection clock re-arms for a fresh cycle.

#### How to resolve a Missing parcel (two flows behave differently)

**This is the #1 QA gotcha:** the action that clears Missing depends on *which marker* fired. A plain re-induction scan clears one flow but **not** the other. Always check the Missing reason first (collection-overdue vs re-induction-overdue).

| Missing reason | Marker | What CLEARS it | What does NOT clear it |
|---|---|---|---|
| **Collection overdue** (never placed in 30 min) | `collection_overdue` | **Place the parcel onto a problem-solve container** (`ShipmentPlacedInContainer`), **or** sort it to position (`ShipmentSortedToPosition` → Resolved), **or** pull it **off a pallet** and re-induct (`ShipmentInductedToSort` with `induction_context = "from_pallet"`) | A normal induction scan (`induction_context = "first"`). Scanning the parcel again at the conveyor does **nothing** — induction ≠ collection. |
| **Re-induction overdue** (placed, not inducted in 20 min) | `locate_package` | **Any induction scan** (`ShipmentInductedToSort`, *either* `first` *or* `from_pallet`) — the workflow cancels `locate_package` at the top of the induction handler regardless of context | — (induction always clears it) |

**Why the difference:**
- `locate_package` means *"the parcel was committed to induction but never showed up at sort"*. The moment it **is** inducted (`ShipmentInductedToSort`, any context), it has physically shown up → marker cancelled → parcel re-derives to its normal status (usually **In progress** if already routed, else New/Placed).
- `collection_overdue` means *"the parcel was never picked up onto a problem-solve cart"*. Scanning it at the induction conveyor does not pick it up onto a cart, so the marker stays. It only clears when the parcel is genuinely **placed into a container**, **sorted** to its final position, or **re-collected off a pallet**.

**Resolution steps for QA:**

| Goal | Collection-overdue Missing | Re-induction-overdue Missing |
|---|---|---|
| Back to **In progress / Placed** | Scan the parcel **into a no-route container** (move-shipment-to-container with `fact.purposes = ["no_route"]`) → `collection_overdue` cleared | Scan the parcel at **sort induction** (any induction scan) → `locate_package` cleared |
| Straight to **Resolved** | Sort the parcel to its final position with the slot confirmed (`scan-audit` with `state.audited = "true"`, or `put-to-bin`, or `confirm-sort-slot`) → `ShipmentSortedToPosition` | same |

> ⚠️ **Common false report:** "I re-scanned the Missing parcel but it's still Missing." This is expected when the Missing reason is **collection overdue** and the scan was an ordinary induction scan (`first`). Induction does not equal collection. To clear it, the parcel must be **placed into a container** (or sorted). Verify the `fact.purposes` on the placement event is an **array** (`["no_route"]`) — if it arrives as a bare string, the translator drops it, `ShipmentPlacedInContainer` is never emitted, and `collection_overdue` never clears.

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
