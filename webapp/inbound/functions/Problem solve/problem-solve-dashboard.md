# Problem Solve Dashboard

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

## 3. Spec / Rules

- The dashboard is **read/observe + direct** — resolution actions (defer, RTS) are initiated by the supervisor, the physical proof comes from worker scans.
- `sort=age` orders the list by time-in-workflow so the oldest/most at-risk shipments surface first.
- A shipment leaves the active board only when it reaches a terminal state (Re-sorted / Deferred / RTS).
- An external **route-reversal** event can re-open a completed (Re-sorted) workflow → the shipment reappears with state **Route Reversed** (see Story 5).
- A **misplaced** flag is raised when expected pallet scans don't all arrive (see Story 7).

## 4. QA / Test notes

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

**Things to watch**
- [ ] Counts on the board match the actual workflow tasks (no double-count when a task transitions).
- [ ] Live update latency — board reflects new scans / routing events without manual refresh.
- [ ] Wrong/unknown `{node}` in the URL → graceful handling (empty/error state, not a crash).
