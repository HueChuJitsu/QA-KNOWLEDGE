# Jitsu QI System — Operational Reference

> **Version 1.0** • Last updated: May 28, 2026
> Source: Internal QI System Proposal v4.0 (May 2026) + Examples
> Effective: **Pilot phase — Driver-Linehaul app (Mobile team)**

A Deshi-powered pipeline that moves quality to the beginning of development — where fixing issues costs the least.

---

## ⚠️ Scope of this document

This doc describes the **new QI System workflow** currently being piloted on the **Driver-Linehaul app**. It is **not yet rolled out** to Mobile (Driver App, Outbound, Warehouse) or Webapp teams — those teams continue to use the old QA model (QMetry, manual verification).

This document is for:
- **Awareness** — all QA members should understand the direction
- **Reference** — for QI Engineers working on Linehaul tickets
- **Onboarding** — pilot members will learn this alongside old-model daily work

For org-wide philosophy and career direction, see [Career Framework](./career-framework/quality-ownership.md).

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| **v1.0** | 2026-05-28 | Initial consolidated reference combining QI System Proposal v4.0 + Deshi command spec + US OAuth2 + Task Biometric examples. Single-source reference for the Mobile/Linehaul pilot. |

---

## Table of Contents

1. [What is the QI System?](#1-what-is-the-qi-system)
2. [Architecture — 3 Layers](#2-architecture--3-layers)
3. [Jira Workflow — 7 States](#3-jira-workflow--7-states)
4. [Entry Paths — 3 Ticket Types](#4-entry-paths--3-ticket-types)
5. [Test Inventory — `tc.md` Format](#5-test-inventory--tcmd-format)
6. [Delta Workflow](#6-delta-workflow)
7. [3 Test Case Types](#7-3-test-case-types)
8. [8 Deshi Commands](#8-8-deshi-commands)
9. [Automation Playbook](#9-automation-playbook)
10. [Example: New US (Empty Inventory)](#10-example-new-us-empty-inventory)
11. [Example: Task (Delta Inventory)](#11-example-task-delta-inventory)
12. [Role Changes](#12-role-changes)
13. [Rollout Phases + Metrics](#13-rollout-phases--metrics)

---

## 1. What is the QI System?

The QI System is a **Deshi-powered pipeline** that shifts quality work to the start of development — where fixing issues is cheapest. It replaces the old "code first → QA tests later" model with a flow where:

- **AC is finalized BEFORE coding** (via Three Amigos)
- **Test cases are generated from inventory** (delta workflow — UPDATE/ADD/KEEP/DEPRECATE)
- **Acceptance tests run on CI** (block merge if fail)
- **Manual TCs are the last gate** (executed by QI in Sign-off)

### Core principle

> Test cases are **evolving system assets** organized by product feature — not disposable outputs of individual tickets. Each feature has its own `tc.md` file forming the **test inventory** = single source of truth.

---

## 2. Architecture — 3 Layers

```
┌──────────────────────────────────────────────────────────┐
│  KNOWLEDGE LAYER                                          │
│  Local repo strategy — $HOME/workspace/gojitsu/           │
│  Deshi auto-syncs (git fetch + pull) before every command│
│  Always fresh code — no stale index                      │
└──────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────┐
│  PROCESS LAYER                                            │
│  PO → QI → Engineering → QS                              │
│  Jira = source of truth during development               │
│  Repo = source of truth after merge (AI agents read here)│
└──────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────┐
│  ENFORCEMENT LAYER (automated, no human trigger)          │
│  Jira workflow gates — block ticket transitions          │
│  CI/CD quality gates — block merge if checks fail        │
└──────────────────────────────────────────────────────────┘
```

### Key insights

- **Dev owns testing**, QS + QI enable it
- **QI defines what to test**, Dev uses Deshi to generate scripts, QS reviews automation quality, QI executes manual cases
- **Source of truth shifts on merge**: Jira during dev → Repo after merge

---

## 3. Jira Workflow — 7 States

```
Draft → In QI Review → Ready for Dev → In Development 
                                            ↓
                                     In QS Review 
                                            ↓
                                     In QI Sign-off → Done
                                            
+ Hotfix (emergency path, separate)
```

| State | Owner | Action to advance |
|-------|-------|------------------|
| **Draft** | PO / Dev / QA / Tech Lead | Create ticket (US/Task/Bug), fill required fields, submit |
| **In QI Review** | QI | US: review-us + Three Amigos + generate TC delta. Task: generate TC delta (review-us optional). Bug: skip — Dev runs generate-testcases after fix, QI reviews on PR |
| **Ready for Dev** | Scrum Master | SM starts sprint |
| **In Development** | Engineering | Read TC Impact Report, pull TC branch, update/add tests using existing helpers, open PR |
| **In QS Review** | Automation team | Review generated scripts and expectations |
| **In QI Sign-off** | Manual QA | Run `/qi:validate-inventory` to verify tc.md matches scripts, execute manual TCs, sign off |
| **Done** | CI Gate | Automatic when all gates pass |
| **Hotfix** | Engineering | Emergency path — retro QI review after deploy |

> **🔁 AC gap loop-back rule**: Any AC gap discovered during development — regardless of size — loops back to **In QI Review**. QI updates AC, generates a new TC delta (UPDATE/ADD/DEPRECATE), updates the test inventory, then forwards back to **In Development**. All changes tracked in Jira + git diff.

---

## 4. Entry Paths — 3 Ticket Types

Different ticket types have different ceremony levels. From TC generation onward, the flow is identical.

| Ticket Type | Created by | PO involved | `/qi:review-us` | Three Amigos | TC delta timing |
|-------------|-----------|-------------|-----------------|--------------|-----------------|
| **User Story** | PO | Yes — owns the story | ✅ Required | ✅ Required (30 min) | Before dev codes |
| **Task** | Dev / QA / Tech Lead | Optional | ⚠️ Optional (only if complex) | Quick sync or skip | Before dev codes |
| **Bug** | Anyone | No | ❌ Skip | ❌ Skip | After fix — Dev runs |

### Bug path — different order

Bugs are dev-first:
1. Bug reported (any source)
2. Dev fixes directly
3. **Dev** (not QI) runs `/qi:generate-testcases` after fix
4. Decision: TC already covers? → KEEP. Not covered? → ADD edge case to prevent regression
5. Dev commits delta + opens PR
6. **QI reviews TC delta on PR** (approve or adjust)
7. From here: QS Review → Sign-off (same as other paths)

---

## 5. Test Inventory — `tc.md` Format

### Repo structure (actual — Linehaul Driver App)

```
line-haul-driver-app/
├── qa/                                    ← QI territory
│   ├── feature_map.yaml                    ← Maps "Affected Apps" → tc.md
│   ├── test-cases/                         ← TEST INVENTORY
│   │   ├── auth_and_account.tc.md          1 feature = 1 file
│   │   ├── route_management.tc.md
│   │   ├── pickup_flow.tc.md
│   │   ├── transit_tracking.tc.md
│   │   ├── dropoff_flow.tc.md
│   │   └── exceptions_and_support.tc.md
│   └── README.md                          ← Automation playbook
│
├── integration_test/                      ← Test scripts (Patrol)
│   ├── acceptance/                        ← By feature
│   ├── e2e/                               ← By feature
│   ├── app_state/, bot/, data/, fixture/, page/, services/, utils/
│
└── test/                                  ← Unit + widget tests
```

### Each TC has these fields

| Field | Purpose | When set |
|-------|---------|----------|
| `id` | Stable TC ID (e.g. `LH-AU-001`) — **never changes, never reused** | Creation — immutable |
| `status` | active / deprecated | Update when deprecated |
| `origin` | Ticket that created this TC (e.g. `MOB-XXXX`) | Creation — immutable |
| `ac_ref` | AC that triggered TC (e.g. `MOB-XXXX/AC-05`) | Creation — immutable |
| `last_updated_by` | Last ticket that modified TC | Every update |
| `script_path` | Path to automation script (if automated) | Updated if file moves |
| `priority` | high / medium | Updated if risk changes |
| `tags` | Feature tags, parallel_safe, serial | As needed |

### TC ID convention

Format: `{APP}-{MODULE}-{NUMBER}`

| Code | Module (for Linehaul) |
|------|---------------------|
| `LH-AU-` | auth_and_account |
| `LH-RM-` | route_management |
| `LH-PU-` | pickup_flow |
| `LH-TT-` | transit_tracking |
| `LH-DO-` | dropoff_flow |
| `LH-EX-` | exceptions_and_support |

> **🚨 ID rule**: When a TC is DEPRECATED, its ID is **NEVER reused**. NUMBER auto-increments per module.

### Example TC entry (markdown format)

```markdown
## LH-AU-001: Linehaul driver logs in successfully via OAuth2

| Field | Value |
|-------|-------|
| Origin | MOB-XXXX |
| AC ref | MOB-XXXX/AC-01 |
| Last updated by | MOB-XXXX |
| Status | active |
| Automated | yes |
| Type | acceptance |
| Priority | high |
| Tags | happy-path, oauth2, login |

**Preconditions:**
- LINEHAUL driver account (enabled, active)
- No active session on any device
- App fresh state, not logged in

| Step | Action | Data | Expected |
|------|--------|------|----------|
| 1 | Open Linehaul Driver app | | Login screen with phone/email/username options |
| 2 | Login with phone number | valid Linehaul driver credentials | Auth succeeds, no error popup |
| 3 | Verify main screen loaded | | Main screen displayed, login screen gone |
| 4 | Verify session state | | Token stored (TTL = 7 days), account enabled |
```

---

## 6. Delta Workflow

### Core rule

> **Deshi MUST NEVER generate test cases from scratch without reading the existing test inventory first.**

This prevents duplicate TCs, ensures obsolete TCs are cleaned up, and keeps inventory accurate over time.

### 4 delta actions

| Action | When | Effect on inventory |
|--------|------|---------------------|
| 🆕 **ADD** | Behavior is new, no existing TC covers it | New entry with new ID |
| 🔄 **UPDATE** | Existing TC needs new steps / changed expectation | Modifies existing entry; `last_updated_by` updated |
| ✅ **KEEP** | Existing TC still covers behavior unchanged | No change |
| 🗑️ **DEPRECATE** | Behavior removed or replaced | Set `status: deprecated`; ID never reused |

### Multi-feature tickets

When a ticket affects more than one feature (e.g. "Add session timeout on pickup screen" → both `auth_and_account` and `pickup_flow`), Deshi reads **ALL affected `tc.md` files** and outputs a delta for each file separately.

### TC Impact Report (auto-posted to Jira)

After running `/qi:generate-testcases`, Deshi posts to Jira:

```
[TC IMPACT] MOB-XXXX — <ticket title>
Feature: <feature_name>
Affected TCs: N

 🆕  ADD     <id>  <description> [type]
 🔄  UPDATE  <id>  <description>
 🗑️  DEPRECATE <id> <description>
 ✅  KEEP    <id>  <description>

Automated: X | Manual: Y
PR: test/MOB-XXXX
```

---

## 7. 3 Test Case Types

| Type | Description | CI gate? | Written by |
|------|-------------|----------|-----------|
| **[ACCEPTANCE]** | One per AC item. Verifies definition of done. | ✅ Blocks merge | Dev via Deshi |
| **[E2E]** | Critical cross-system flows only (inbound → routing → delivery → billing) | Staging post-merge | Dev |
| **[MANUAL]** | "Can be automated: No". Complex judgment, UX, exploratory cases | No (manual gate) | QI executes in Sign-off |

➕ Unit tests already exist separately.

> **📌 QMetry NOT in scope for pilot** — repo (`tc.md`) is the source of truth. CSV export to QMetry is possible later if needed, but repo remains authoritative.

---

## 8. 8 Deshi Commands

### Pipeline commands (every ticket)

| Command | Actor | When to use |
|---------|-------|-------------|
| `/po:review-us --jira MOB-XXXX` | Ticket creator | Self-check before submitting — mainly US, optional for Task/Bug |
| `/qi:review-us --jira MOB-XXXX` | QI Engineer | Deep analysis — required for US, optional for Task, **skip for Bug** |
| `/qi:generate-testcases --jira MOB-XXXX` | QI (or Dev for Bug tickets) | Read inventory → generate TC delta → update `*.tc.md` + PR branch + Impact Report |
| `/qs:generate-tests --jira MOB-XXXX` | Developer | Generate Acceptance + E2E scripts from TC list. MANUAL TCs skipped automatically |
| `/qs:review-test-quality --pr --jira MOB-XXXX` | QS Engineer | Review test script quality and coverage on the PR |
| `/qi:validate-inventory --jira MOB-XXXX` | QI Engineer | Semantic check: `tc.md` steps match actual script logic. Run before Sign-off to catch drift |

### Infrastructure commands

| Command | Actor | Purpose |
|---------|-------|---------|
| `/qs:sync-repos` | QS / CI | Force sync all repos under `$HOME/workspace/gojitsu/` |
| `/qs:build-event-map` | QS Engineer | Aggregate cross-repo event fanout map from local repos |

### `/qi:review-us` output format

Deshi posts a comment with structured findings:

```
[QI REVIEW] MOB-XXXX — <ticket title>
── BLOCKERS ──
  BLOCKER <issue>
    Ticket says: <quote>
    Action: <what needs to happen>

── CLARIFICATIONS NEEDED ──
  CLARIFY <issue>
    Suggestion: <recommendation>

── EDGE CASES ──
  EDGE CASE <scenario>

── POSITIVE NOTES ──
  GOOD <what's done well>

── RECOMMENDATION ──
  Status: NEEDS REVISION / READY
```

### Required shared infrastructure

5 things MUST exist before Deshi commands work:
1. **`feature_map.yaml`** — maps Jira "Affected Apps" → `tc.md` files + script folders
2. **TC ID convention** — `{APP}-{MODULE}-{NUMBER}` (see Section 5)
3. **Empty `tc.md` files** — auto-created by Deshi if not yet present
4. **Jira custom fields** — Affected Apps, US Type, QI Review Status
5. **Permissions** — Jira API (read/write) + Git/repo access (read tc.md/data/scripts, write tc.md + acceptance scripts, create branches + PRs)

---

## 9. Automation Playbook

Rules for AI (Deshi) and Dev when writing or updating test scripts.

### AI workflow — mandatory order

1. **Read Jira ticket** → identify feature + behavior change
2. **Read test inventory** → find existing TCs for affected features
3. **Determine delta** → ADD / UPDATE / KEEP / DEPRECATE
4. **Generate/update scripts** → using existing helpers (`page/services/fixture/utils`)
5. **Update inventory** + post TC Impact Report

### Script rules

| Rule | Why |
|------|-----|
| Prefer **UPDATE** existing test over **ADD** new test | Prevents duplicate TCs covering same behavior |
| Do NOT write setup logic inside test scripts | Use shared helpers (`page/services/fixture`). Scripts only orchestrate + assert |
| Every test must map to a TC ID | Traceability: `LH-AU-001` in tc.md ↔ test function in script |
| Every created entity must be cleaned up in `tearDown` | Tests that leak data break other tests |
| Do NOT duplicate existing helper logic | If `services/` already has the method, use it |
| Read `page/`, `services/`, `fixture/`, `utils/` before writing new helpers | Prevents 3 versions of "login driver" |
| Rename or move a test file → update `script_path` in `*.tc.md` | CI check validates all `script_path` entries point to existing files |

### Setup & data rules

| Rule | Example |
|------|---------|
| TC data column describes WHAT data is needed in natural language | TC says "valid Linehaul driver credentials" → AI reads `data/` folder to find matching file |
| Use support helpers for API/auth setup | `auth.loginDriver(phone: profile.driverPhone, otp: profile.otp)` |
| Cleanup in `tearDown` | Reset test account state, clear session tokens |
| Use `testRunId` for data isolation when available | Prevents parallel test collision on shared test accounts |

### Parallel & isolation rules

| Situation | Rule |
|-----------|------|
| Test only reads data, never modifies shared state | Tag `parallel_safe: true` — can run in parallel |
| Test modifies shared accounts, session state, or global config | Tag `parallel_safe: false` + `@serial` — must run alone |
| Test modifies global config or shared test account | Must restore original state in `tearDown` |

### Do NOT rules (for AI)

```
AI must NOT:
  ✗ Generate tests from scratch without reading inventory
  ✗ Create duplicate TCs for behavior already covered
  ✗ Write inline setup when a helper exists
  ✗ Skip cleanup / tearDown
  ✗ Create new helpers without checking page/services/fixture/utils/ first
  ✗ Ignore existing tests when a ticket changes behavior
  ✗ Create folders by Jira ticket ID
```

---

## 10. Example: New US (Empty Inventory)

**Ticket**: `MOB-XXXX` "Jitsu OAuth2 Login with DSP Type Gating for Linehaul Driver App"
**Type**: User Story (created by Tech Lead)
**Affected App**: Driver Linehaul → feature `auth_and_account`
**Starting state**: Inventory empty (first US for this feature)

### Step 1 — `/qi:review-us`

Deshi reads ticket + linked tickets + repo context, posts comment with structured findings.

**Output captured**:
- **2 BLOCKERS**: Token TTL undefined ("7 days???"), Account lockout threshold N undefined
- **4 CLARIFICATIONS**: Username max length, "Clear cache" scope, environment switcher visibility, DSP rejection message text
- **3 EDGE CASES**: Backend unreachable during auth, DSP type change mid-session, all login methods same gating?
- **4 POSITIVE NOTES**: Success criteria clear, session rules defined, MOB-2543 spec referenced, DSP types explicit
- **RECOMMENDATION**: NEEDS REVISION — resolve blockers before Three Amigos

### Step 2 — Three Amigos (30 min)

PO + QI + Tech Lead resolve findings:
- Token TTL: **7 days** (configurable)
- Lockout: **5 attempts → 30 min**
- Username max: **50 chars** default
- "Clear cache": clears auth tokens + cached profile (NOT offline queue)
- Logo 7-tap: debug builds only
- Rejection message: "This app is for Linehaul drivers only. Contact your dispatcher."
- Backend unreachable: "Unable to connect. Check your internet connection."
- DSP mid-session change: **out of scope**, create follow-up
- All 3 login methods: same gating applies

**Result**: 11 finalized ACs written to Jira (AC-01 through AC-11)

### Step 3 — `/qi:generate-testcases`

Deshi reads ACs + inventory (empty) → generates **10 TCs** (all ADD):

| TC ID | Description | Type |
|-------|-------------|------|
| LH-AU-001 | Linehaul driver login happy path | acceptance |
| LH-AU-002 | Non-Linehaul driver blocked at backend | acceptance |
| LH-AU-003 | Non-existing account error | acceptance |
| LH-AU-004 | Login field validations | acceptance |
| LH-AU-005 | Force logout multi-device | **manual** (2 devices needed) |
| LH-AU-006 | Auto-login with valid token | acceptance |
| LH-AU-007 | Expired token → login screen | acceptance |
| LH-AU-008 | Account lockout after 5 fails | acceptance |
| LH-AU-009 | Backend unreachable | acceptance |
| LH-AU-010 | Login screen UI elements | acceptance |

**Automated: 9 | Manual: 1 (LH-AU-005)**

### Summary — value of the flow

| Without QI System | With QI System |
|-------------------|----------------|
| Dev interprets "Account lockout after N login fail ???" themselves | Ambiguity caught at Step 1, resolved at Step 2 — **before any code written** |
| Test cases written from scratch each ticket | TCs generated with traceability to AC |
| QA finds AC gaps at end of sprint | Issues caught upfront, sprint focused on building |

---

## 11. Example: Task (Delta Inventory)

**Ticket**: `MOB-YYYY` "Add biometric login after first successful OAuth2 login"
**Type**: Task (created by Tech Lead)
**Affected App**: Driver Linehaul → feature `auth_and_account`
**Starting state**: Inventory has **10 active TCs** (from previous OAuth2 work)

### Step 1 — Tech Lead creates ticket (10 min)

AC written by Tech Lead, technical + specific (AC-01 → AC-06).

### Step 2 — QI decides: skip `/qi:review-us` (2 min)

Why skip?
- Type = Task → review-us is optional
- AC written by Tech Lead → already technical
- Single feature affected
- No ambiguity in AC

### Step 3 — `/qi:generate-testcases` (2 min — Deshi does work)

Deshi reads existing 10 TCs + new AC → generates **delta**:

| Action | Count | TCs |
|--------|-------|-----|
| 🔄 UPDATE | 3 | LH-AU-001, LH-AU-006, LH-AU-010 |
| 🆕 ADD | 4 | LH-AU-011 (skip biometric), LH-AU-012 (biometric login), LH-AU-013 (biometric fails → fallback), LH-AU-014 (device without biometric) |
| ✅ KEEP | 7 | LH-AU-002, 003, 004, 005, 007, 008, 009 |

**Before**: 10 TCs → **After**: 14 TCs

### Step 4 — QI reviews delta (10 min)

QI checks delta makes sense, edits minor things (e.g. add specific assertion to LH-AU-014), advances ticket → **Ready for Dev**.

### Step 6 — Dev codes + runs `/qs:generate-tests`

Deshi reads TC Impact Report → knows 3 scripts need updating, 4 new scripts needed. Generates DRAFT scripts.

Dev adjusts ~20% (e.g. fixes biometric mock to use `flutter_biometrics`, creates `#biometricToggle` widget key in feature code).

### Steps 7-9 — QS Review → QI Sign-off → CI Gate

QS reviews test quality on PR. No manual TCs for this ticket (LH-AU-005 manual TC unrelated). CI gates pass. PR merged → Done.

### Total time

| Step | Who | Time |
|------|-----|------|
| Create ticket | Tech Lead | 10 min |
| Decide review-us | QI | 2 min |
| Generate TC delta | QI + Deshi | 2 min |
| Review delta | QI | 10 min |
| Quick sync (optional) | QI + Dev | 0-5 min |
| Code + tests | Dev + Deshi | 1-2 hrs |
| QS review | QS | 20 min |
| QI sign-off | QI | 5 min |
| **Total** | | **~2.5 hrs** |

### Key learnings

- **Deshi generates 70-80% correctly**, Dev completes the remaining 20-30%
- Dev never writes tests from scratch — always builds on existing helpers + tc.md
- TC IDs traced 1:1 to scripts — no orphans, no duplicates
- Manual TCs untouched unless ticket directly affects them

---

## 12. Role Changes

| Role | Today (Old Model) | With QI System |
|------|-------------------|----------------|
| **PO** | Writes US, issues caught late | Creates US with full AC. Self-checks via `/po:review-us` before submitting. Joins Three Amigos. |
| **QI / Manual QA** | Joins late, catches bugs manually | Defines quality upfront (Round 1: review-us + Three Amigos + TC delta) + executes manual TCs before release (Round 2: Sign-off) |
| **Developer** | Ambiguous tickets, interprets AC themselves | Reads TC Impact Report, pulls TC branch, updates/adds tests via Deshi using existing helpers — **does not write tests from scratch** |
| **QS / Automation** | Writes scripts separately, reacts to bugs | Reviews generated scripts and expectations — **enables dev, does not write during sprints** |
| **Scrum Master** | Manages sprint ceremony | Gates **Ready for Dev** — starts sprint to activate tickets for dev |

### QI team capacity allocation

| Activity | Allocation | What it includes |
|----------|-----------|------------------|
| Ceremonies + collaboration | ~30% | Backlog refinement, Three Amigos, sprint planning, review, retro |
| Actual work | ~70% | AC writing (manual), TC delta generation (inventory-aware), manual TC execution in Sign-off |

> **📌 Sprint cadence**: Backlog refinement happens during Sprint N to prepare tickets for Sprint N+1 — QI is always working one sprint ahead on refinement while executing Sign-off for current sprint.

---

## 13. Rollout Phases + Metrics

### 3 Rollout Phases

| Phase | Scope | Status |
|-------|-------|--------|
| **Phase 1** | Build framework — Deshi commands, repo setup, Patrol integration. Mobile pilot foundation. | Active build |
| **Phase 2** | Pilot pipeline — Driver-Linehaul (new app, clean slate). Validate full process end-to-end with no legacy complexity. | Active pilot |
| **Phase 3** | Scale + backlog — Full org rollout. Separate initiative for existing code coverage. **Target: 80% org-wide coverage by end of year.** | Future |

### Expected metrics (6-month targets)

| Metric | Now | Target |
|--------|-----|--------|
| Issues caught BEFORE dev starts | Rare | **>60%** at QI Review stage |
| Tickets with complete AC before sprint | ~30% | **>90%** |
| Acceptance test coverage per ticket | 0% | **100%** of AC items |
| Manual TCs executed before release | Inconsistent | **100%** of non-automatable TCs |
| Test coverage org-wide | Unknown | **Trending up, never decreasing** |
| Rollback from policy conflicts | Occasional | **Zero** |
| Release stress | High — concentrated | **Low — continuous, automated** |

---

## Related Documents

- [Career Framework](./career-framework/quality-ownership.md) — Quality Ownership, QI Engineering, QS Engineering ladders
- [Probation Goals](./jitsu-qa-probation-goals.md) — Current Junior/Senior probation framework
- [Web App Release Process](./jitsu-qa-webapp-release-process-full-guide.md) — Current release SDLC (parallel system, applies to non-Linehaul teams)

---

*Source materials: QI System Proposal v4.0, OAuth2 US example, Biometric Task example, Deshi build spec (internal, May 2026).*
