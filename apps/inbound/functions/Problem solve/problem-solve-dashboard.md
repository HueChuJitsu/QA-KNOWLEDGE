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
| `MISSING` | Missing | `locate_package` task is live — misplaced-detection fired, action required | **Step 1 (start timer):** `ContainerCommittedToSortInduction` arrives for the container this shipment was placed into → workflow schedules a **20-minute countdown** anchored to the event's `committed_at`<br>**Step 2 (timeout fires):** shipment does NOT appear at induction within 20 min → timeout fires → `locate_package` task spawned → status becomes `MISSING`<br>**Cancelled if:** `ShipmentInductedToSort` arrives before timeout — timer is cancelled, `locate_package` is never spawned | `locate_package` task active; overrides all non-terminal states |
| `RESOLVED` | Resolved | Workflow is terminal — problem solve closed | **Path A:** `ShipmentSortedToPosition` — completes `sort_to_position` task, phase = `sorted`, workflow terminal<br>**Path B:** `TriggerManualAction(RTS)` — phase = `rts_exit`, workflow terminal<br>**Path C:** `WorkflowInstanceAborted` — force-aborted via admin RPC | `is_terminal = true` |

> `new` parcels are not broken out as a tile in the Open Problem Solves hero card. Derive the count as: `totalOpen − missing − inProgress − placed`.
