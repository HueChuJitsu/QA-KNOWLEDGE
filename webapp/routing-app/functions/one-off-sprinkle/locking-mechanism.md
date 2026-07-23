# Locking Mechanism: Sprinkle / Redelivery / Add-shipment

> Related ticket: [ALT-1881](https://gojitsu.atlassian.net/browse/ALT-1881) — Implement locking mechanism during automatic shipment sprinkling
> Source: code read from the `dev` branch (2026-07-22) across 5 repos — `dao`, `data-orchestrator`, `sortation-bizlogic`, `inbound-api-grpc`, `sprinkle-bizlogic` — cross-checked against real Datadog logs.

## General idea

When the system wants to touch **a shipment** to process it, it "reserves" that shipment with a temporary lock for a few seconds up to ~60 seconds. While the lock is held, **no other action is allowed to touch the same shipment at the same time**. Once processing finishes, the lock is released right away.

Three actions commonly compete to process the same shipment:

1. **Auto-sprinkle** (`oneOffSprinkleV2`) — the system automatically assigns a shipment onto an existing route the moment it's scanned at the warehouse, with no human action involved.
2. **Add single shipment to redelivery** (`RedeliveryManager.addShipment`) — a staff member/dispatcher manually adds a shipment to redelivery (re-assigns it to a route).
3. **Bulk redelivery / API v3** (`RedeliveryManagerV2.redeliveryMultiShipment`) — same as #2 but for many shipments at once. HTTP endpoint: `POST redelivery/redelivery/multiple/v3` (service `dataorch`).

There's also **"Select Solution"** (dispatcher/admin approving a whole new batch of routes for a region in Routing Admin) — this uses a separate lock, rarely relevant to day-to-day testing; see section 4.

---

## 1. Lock per SHIPMENT (most important for QA)

- Shared across all 3 actions: auto-sprinkle, add single shipment, bulk redelivery — they all fight over the same "reservation book" (key: `REDELIVERY:SH_<shipmentId>`, TTL ~60s, implemented in the shared `dao` library).
- If a shipment is already being held by one action, another action calling **at the same time** gets rejected — two actions can never process the same shipment simultaneously.

**What each blocked action actually returns:**

| Blocked action | Response returned | Shows up in Datadog? |
|---|---|---|
| Add single shipment | HTTP 400, message: `"Shipment is being added as redelivery"` | ✅ Yes — see "Checking Datadog" below |
| Bulk redelivery (API v3) | That shipment's entry in the response has `"message": "Shipment is updating"`; the rest of the batch still processes normally (overall HTTP 200) | ❌ No — it only exists in the JSON response, it's never logged |
| Auto-sprinkle | No error surfaced anywhere (it's automatic, no human waiting on it) — the system logs it and skips that shipment for this attempt; it will be retried on the next scan/attempt | ✅ Yes |

---

## 2. Lock per ROUTE (assignment)

- Besides the per-shipment lock, the system also locks the specific **route/assignment** the shipment is being placed onto.
- Shared only between **auto-sprinkle** and **bulk redelivery** (single-shipment add does NOT use this lock).
- Held very briefly, just a few seconds (3–30s) — just long enough to check/modify that route.
- If the route is already locked, the other action skips it:
  - Bulk redelivery returns `"message": "Assignment is updating"` for shipments targeting that route.
  - Auto-sprinkle: that route is excluded from the candidate list for this attempt.

**Why a separate lock for the route:** even if two different shipments are involved, if both are competing for the **same route**, it still has to be blocked — otherwise the route could get overloaded, or two processes could modify it at once and corrupt data.

---

## 3. Lock summary table

| Lock type | Key | TTL | Used by |
|---|---|---|---|
| Per shipment | `REDELIVERY:SH_<shipmentId>` | ~60s | Auto-sprinkle, Add single shipment, Bulk redelivery |
| Per route (one-off) | `LOCK-ONE-OFF-SPRINKLE-<assignmentId>` | 3–30s | Auto-sprinkle, Bulk redelivery |

---

## 4. Third lock type (rarely relevant to daily testing): "Select Solution"

When a dispatcher/admin clicks **"Select Solution"** to approve a whole new batch of routes for a region in Routing Admin, the system locks that entire region while it's being approved (separate keys: `sprinkling-lock-<assignmentId>` and a zone lock `sprinkling-lock-<region>-<zone>`), so auto-sprinkle can't jump in and modify routes that are currently being approved.

QA typically only encounters this when testing the **Routing Admin** screen — it's unrelated to day-to-day redelivery/sprinkle test cases.

---

## 5. Checking Datadog

⚠️ **The correct service tag is `dataorch`**, not `data-orchestrator`.

### Can be found in Datadog

The **single-shipment add blocked by lock** case — has a real log entry (level `error`):

```
service:dataorch "Cannot obtain lock for redelivery shipment ID <id>"
```

The **auto-sprinkle blocked by the route/assignment lock** case also has a real log entry — service `inbound-api`, not `dataorch`, logged by `SprinklingManager`/`SortInfoManager` (thread context is whatever triggers a scan, e.g. `GET /info/<shipmentId>`, since auto-sprinkle runs as a side-effect of that call rather than its own endpoint):

```
service:inbound-api "Error while sprinkling shipment ID <shipmentId>"
service:inbound-api "com.axlehire.sortation.bizlogic.exception.RetryableException: Failed to lock for assignment <assignmentId>"
```

Stack trace confirms the code path: `SprinklingManager.doOneOffSprinkleV2` → `SprinklingManager.oneOffSprinkleV2WithRetry` → `SortInfoManager.lambda$scanInbound$38`. Before this final failure, look for `warn`-level `retry one off sprinkling for shipment <id> at <region>` lines — the retry loop runs a few times (observed: 3 retries, ~150-250ms apart) before giving up and logging the error above.

### Cannot be found in Datadog

The two bulk-redelivery messages — `"Shipment is updating"` and `"Assignment is updating"` — **zero results over the last 30 days**, verified directly in Datadog. Reason: the code only attaches these two strings to the JSON response returned to the API caller (`RedeliveryManagerV2.java`); no log statement ever writes them out.

→ To verify the bulk "is updating" case, you must inspect the actual **API response JSON** directly (Postman / client-side request-response log) — it cannot be found via Datadog logs.

---

## 6. Suggested race-condition test cases for QA

1. **Add the same shipment twice, almost simultaneously** (fire two requests back-to-back very quickly) → only one request should succeed; the other must get exactly `"Shipment is being added as redelivery"`, and no duplicate redelivery should be created for the same shipment.
2. **Bulk redelivery on a batch of 10 shipments, where one is being processed by auto-sprinkle at the same time** → that one shipment must come back as `"Shipment is updating"` in the response, while the rest of the batch still proceeds normally (the whole batch must not be blocked).
3. **Two different shipments competing for the same route** (e.g. a nearly-full route) → only one shipment gets placed; the other should get `"Assignment is updating"` or get tried against a different route.
4. After testing (whether it succeeded or was rejected), check that no shipment is left in a **half-finished state** (e.g. redelivery info created but the route wasn't updated, or vice versa) — that's a sign the lock was "forgotten" and never released.

---

## 7. References (PR/code)

| Repo | File | Notes |
|---|---|---|
| `dao` | `LockDAO.java`, `LockRD.java` | Shared shipment-level lock, PR [#2081](https://github.com/gojitsucom/dao/pull/2081) |
| `data-orchestrator` | `RedeliveryManager.java`, `RedeliveryManagerV2.java` | Single-shipment add + bulk redelivery, PR [#2768](https://github.com/gojitsucom/data-orchestrator/pull/2768) |
| `data-orchestrator` | `AssignmentManager.java` | Route-level lock (`LOCK-ONE-OFF-SPRINKLE-`) used by bulk redelivery |
| `sortation-bizlogic` | `SprinklingManager.java` | Auto-sprinkle (`oneOffSprinkleV2`) — shares both the shipment lock and the route lock |
| `sprinkle-bizlogic` | `SprinklingCacheManager.java`, `SprinklingSolutionSelector.java` | Zone lock for "Select Solution" |

⚠️ **Note on this reaching production**: as of the time this was checked (2026-07-22), `data-orchestrator` on the `dev` branch depends on an unreleased SNAPSHOT of `dao` (`1.0.56-ALT-130-lock-SNAPSHOT`, built from the `release/ALT-130-lock` branch). If lock behavior doesn't match what's described here, it could be because the build under test isn't using the right `dao` version — flag it to a developer to confirm before concluding it's a bug.
