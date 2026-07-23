---
# ===== IDENTITY =====
feature: one-off-sprinkle-locking-mechanism
title: Locking Mechanism — Sprinkle / Redelivery / Add-shipment
domain: routing-app
sub_domain: one-off-sprinkle
schema_version: 1.0

# ===== STATE =====
status: active
maturity: evolving                      # dev branch depends on an unreleased dao SNAPSHOT — see §6
maintainer: huong.pham

# ===== SEARCH & DISCOVERY =====
keywords:
  - locking mechanism
  - lock
  - race condition
  - one-off sprinkle
  - auto-sprinkle
  - redelivery
  - bulk redelivery
  - shipment lock
  - route lock
  - assignment lock
  - ALT-1881

# ===== TAXONOMY =====
user_types:
  - Dispatcher / Admin
  - Warehouse staff
system_touchpoints:

# ===== EXTERNAL SOURCES =====
confluence_refs: []

figma_refs: []
---

# Locking Mechanism — Sprinkle / Redelivery / Add-shipment

> **Scope** — The per-shipment and per-route locks that stop **auto-sprinkle**, **add single shipment to redelivery**, and **bulk redelivery (API v3)** from processing the same shipment or route at the same time. Out of scope: the separate region/zone lock used by **"Select Solution"** in Routing Admin — mentioned only for context in §5.

> Source: code read from the `dev` branch (2026-07-22) across 5 repos — `dao`, `data-orchestrator`, `sortation-bizlogic`, `inbound-api-grpc`, `sprinkle-bizlogic` — cross-checked against real Datadog logs.

---

## 1. Actors & Systems

- **Auto-sprinkle** (`oneOffSprinkleV2`) — system automatically assigns a shipment onto an existing route the moment it's scanned at the warehouse; no human action involved.
- **Add single shipment to redelivery** (`RedeliveryManager.addShipment`) — staff/dispatcher manually adds one shipment to redelivery (re-assigns it to a route).
- **Bulk redelivery / API v3** (`RedeliveryManagerV2.redeliveryMultiShipment`) — staff/dispatcher does the same as above for many shipments at once. Endpoint: `POST redelivery/redelivery/multiple/v3` (service `dataorch`).
- **Select Solution** (dispatcher/admin, Routing Admin) — approves a whole new batch of routes for a region; uses a separate lock, rarely relevant to day-to-day testing (§5).
- **Lock keys** (shared `dao` library, cache-based reservations — not DB tables) — see §4 for the exact keys and TTLs.

---

## 2. Business Rules

### 2.1. Per-shipment lock (most important for QA)

**Block when ANY condition is true:**

| # | Condition |
|---|---|
| B1 | Another action already holds `REDELIVERY:SH_<shipmentId>` for this shipment (TTL ~60s) |

**Core rule**: Two actions can never process the same shipment at the same time — whichever action acquires `REDELIVERY:SH_<shipmentId>` first wins; the other is blocked outright, not queued.

### 2.2. Per-route (assignment) lock

**Block when ANY condition is true:**

| # | Condition |
|---|---|
| B1 | Auto-sprinkle or bulk redelivery already holds `LOCK-ONE-OFF-SPRINKLE-<assignmentId>` for the target route (TTL 3–30s) |

**Core rule**: Even two *different* shipments competing for the same route are serialized. Shared only between **auto-sprinkle** and **bulk redelivery** — single-shipment add does not use this lock at all.

### 2.3. Special Cases

- **Select Solution** uses a separate, region-scoped lock (`sprinkling-lock-<assignmentId>` and zone lock `sprinkling-lock-<region>-<zone>`) so auto-sprinkle can't modify routes while a batch is being approved. QA typically only encounters this when testing the Routing Admin screen — see §5.

---

## 3. Workflow

1. An action (auto-sprinkle / add single shipment / bulk redelivery) attempts to acquire the per-shipment lock `REDELIVERY:SH_<shipmentId>`.
2. If it's already held by another action → blocked immediately; the response differs per action (see table below). Single-add and bulk redelivery do not retry; auto-sprinkle retries on the next scan/attempt.
3. If acquired, **auto-sprinkle** and **bulk redelivery** additionally attempt to acquire the per-route lock `LOCK-ONE-OFF-SPRINKLE-<assignmentId>` for the target route (single-shipment add skips this step — see §2.2).
4. If the route lock is already held → that route is skipped/blocked for this attempt only (§2.2), the shipment lock from step 1 is still released normally.
5. The action processes the shipment/route.
6. Both locks are released immediately once processing finishes.

**What each blocked action actually returns (step 2):**

| Blocked action | Response returned | Shows up in Datadog? |
|---|---|---|
| Add single shipment | HTTP 400, message: `"Shipment is being added as redelivery"` | ✅ Yes — see §6 Gotchas |
| Bulk redelivery (API v3) | That shipment's entry in the response has `"message": "Shipment is updating"`; the rest of the batch still processes normally (overall HTTP 200) | ❌ No — only exists in the JSON response, never logged |
| Auto-sprinkle | No error surfaced anywhere (fully automatic, no human waiting) — the system logs it and skips that shipment for this attempt; retried on the next scan | ✅ Yes |

**What happens when the route lock (step 3/4) is held:**

- Bulk redelivery returns `"message": "Assignment is updating"` for shipments targeting that route.
- Auto-sprinkle: that route is excluded from the candidate list for this attempt.

---

## 4. Data Model

> These are cache-based lock reservations (shared `dao` library), not database tables.

### Lock keys

| Lock type | Key | TTL | Used by |
|---|---|---|---|
| Per shipment | `REDELIVERY:SH_<shipmentId>` | ~60s | Auto-sprinkle, Add single shipment, Bulk redelivery |
| Per route (one-off) | `LOCK-ONE-OFF-SPRINKLE-<assignmentId>` | 3–30s | Auto-sprinkle, Bulk redelivery |
| Region (Select Solution) | `sprinkling-lock-<assignmentId>` | — | Select Solution |
| Zone (Select Solution) | `sprinkling-lock-<region>-<zone>` | — | Select Solution |

---

## 5. Related Behaviors

- **Select Solution** (Routing Admin): when a dispatcher/admin approves a whole new batch of routes for a region, the system locks that entire region (keys above) so auto-sprinkle can't jump in and modify routes currently being approved. Unrelated to day-to-day redelivery/sprinkle test cases.
- **Auto-sprinkle retry loop**: when the route lock can't be acquired, `SprinklingManager` retries a few times (observed: 3 retries, ~150–250ms apart, `warn`-level `retry one off sprinkling for shipment <id> at <region>` logs) before giving up and logging the final error (see §6).
- **One-off sprinkle data/config/scan flow**: see [One-Off Sprinkle — Data Setup, Config Checks & Scan Flow](../../sprinkling/one-off-sprinkle-guide.md) for how a shipment enters the sprinkle flow in the first place; this doc only covers the locking layer once that flow is triggered.

---

## 6. Known Issues, Gotchas & Ambiguities

### Currently open issues

- **Production readiness**: as of 2026-07-22, `data-orchestrator` on the `dev` branch depends on an unreleased SNAPSHOT of `dao` (`1.0.56-ALT-130-lock-SNAPSHOT`, built from `release/ALT-130-lock`). If observed lock behavior doesn't match this doc, confirm the build under test is using the right `dao` version with a developer before concluding it's a bug.

### Gotchas

- **Datadog service tag**: the correct tag is `dataorch`, **not** `data-orchestrator`.
- **Bulk redelivery messages are invisible in Datadog**: `"Shipment is updating"` and `"Assignment is updating"` exist only in the API response JSON (`RedeliveryManagerV2.java`) — zero results over the last 30 days, verified directly in Datadog. No log statement ever writes them out. To verify these cases, inspect the actual **API response JSON** (Postman / client-side request-response log) directly.
- **Single-shipment add blocked by the shipment lock**: logged with a real entry (level `error`), service `dataorch`:
  ```
  service:dataorch "Cannot obtain lock for redelivery shipment ID <id>"
  ```
- **Auto-sprinkle blocked by the route/assignment lock**: logged under service `inbound-api` (not `dataorch`), by `SprinklingManager`/`SortInfoManager`. Thread context is whatever triggers the scan (e.g. `GET /info/<shipmentId>`), since auto-sprinkle runs as a side-effect of that call rather than its own endpoint:
  ```
  service:inbound-api "Error while sprinkling shipment ID <shipmentId>"
  service:inbound-api "com.axlehire.sortation.bizlogic.exception.RetryableException: Failed to lock for assignment <assignmentId>"
  ```
  Stack trace confirms the code path: `SprinklingManager.doOneOffSprinkleV2` → `SprinklingManager.oneOffSprinkleV2WithRetry` → `SortInfoManager.lambda$scanInbound$38`.

### Fixed issues (kept for regression coverage)

- None currently.

### Open ambiguities

- None currently.

---

## 7. Suggested Race-Condition Test Cases

| # | Test case | Expected result |
|---|---|---|
| TC1 | Add the same shipment twice, almost simultaneously (fire two requests back-to-back very quickly) | Only one request succeeds; the other gets exactly `"Shipment is being added as redelivery"`. No duplicate redelivery is created for the same shipment. |
| TC2 | Bulk redelivery on a batch of 10 shipments, where one is being processed by auto-sprinkle at the same time | That one shipment comes back as `"Shipment is updating"` in the response; the rest of the batch still proceeds normally (the whole batch is not blocked). |
| TC3 | Two different shipments competing for the same route (e.g. a nearly-full route) | Only one shipment gets placed; the other gets `"Assignment is updating"` or is tried against a different route. |
| TC4 | After testing (whether it succeeded or was rejected), inspect the shipment/route state | No shipment is left half-finished (e.g. redelivery info created but the route wasn't updated, or vice versa) — that would indicate a lock was "forgotten" and never released. |

---

## 8. References

### Related tickets

- [ALT-1881](https://gojitsu.atlassian.net/browse/ALT-1881) — Implement locking mechanism during automatic shipment sprinkling


### Related features

- [One-Off Sprinkle — Data Setup, Config Checks & Scan Flow](../../sprinkling/one-off-sprinkle-guide.md)

### Confluence

- None currently.

### Glossary

- None currently.
