# Jitsu QA — Two-Week Onboarding Plan

**JITSU QA TEAM**

*Day 1 → Day 14 Learning Roadmap*

**🎯 DESIGNED FOR**

- ✓ Junior–Mid QA Engineers (4–5 years experience)
- ✓ Already familiar with web & mobile QA fundamentals
- ✓ New to logistics / last-mile delivery domain
- ✓ Intensive pace — ready for live tickets by Day 7–10

*Use this plan alongside the Jitsu QA Training Document.*

*Version 1 — May 2026*

---

## How to Use This Plan

This roadmap is your day-by-day guide for the first two weeks at Jitsu. It assumes you already know QA fundamentals — the focus here is the Jitsu domain, tooling, and team workflow.

### Daily Structure

Each day is broken into 3 blocks:

| Block | What you do |
| --- | --- |
| 🌅 **Morning (3–4h)** | Learning block — read docs, follow tutorials, watch demos. Focus on absorbing context. |
| 🛠 **Afternoon (3–4h)** | Hands-on practice — log into systems, run flows on Staging, write/run test cases. |
| ✅ **End-of-day (30m)** | Reflection: write daily report, note questions for QA lead, check tomorrow's plan. |

### Tracking Progress

- Every day has a checklist (☐). Tick when done.
- Each day ends with a 'Daily Slack Report' — a quick update in the QA channel.
- Each week ends with a milestone check-in with your QA lead.

> **🎯 SUCCESS CRITERIA**
> By the end of Day 14, you should be able to: pick up a real maintain ticket independently → write test cases from the End User viewpoint → execute on Staging/Beta → write a verification comment on Jira → report status in the QA channel.

### If You Fall Behind

- This plan is intensive — it's normal to feel overwhelmed Days 1–3.
- If a day's tasks run over, just report progress honestly in Slack. Catch up on missed reading on weekends if needed.
- Block 30 minutes daily for questions with your QA lead/buddy.

### 📢 Daily Slack Report Format

Post your daily report in the QA channel at the end of each day. Keep it short — 4–6 lines is enough. Use this template:

```
📅 Day [N] — [Date]

✅ Done:
- [What you finished today]

📚 Learned:
- [Key concept or insight]

❓ Blockers / Questions:
- [Anything stuck / need help with]

🎯 Tomorrow:
- [Brief plan for next day]
```

> **💡 WHY SLACK ONLY?**
> Daily reports in Slack keep the team in the loop without overhead. Your QA lead can scan quickly, jump in if you're stuck, and track your progress naturally. No extra docs to maintain.

---

## Before Day 1 — Prerequisites

*Get these set up before your first day. If something is blocked, flag it to HR/IT immediately.*

### 📧 Accounts Required

- ☐ Jitsu email (@jitsu / @gojitsu / @axlehire)
- ☐ **Drata account** (drata.com) — data & device security management, login with Jitsu Google account
- ☐ **1Password account** (1password.com) — shared credentials/secrets vault, login with Jitsu Google account
- ☐ Jira access (gojitsu.atlassian.net)
- ☐ Confluence access (gojitsu.atlassian.net/wiki)
- ☐ QMetry access (test management tool)
- ☐ Slack workspace + add to qa-mobile, qa-web channels
- ☐ Google Drive access (for test case templates)
- ☐ Datadog access (for log monitoring)
- ☐ **StrongDM account** (app.strongdm.com) — gateway for DB, Jenkins, Consul, Nexus access
- ☐ Database resources approved in StrongDM: PostgreSQL + MongoDB (Staging + Prod read-only)
- ☐ RabbitMQ access (for testing workers)
- ☐ Staging accounts on each system (Dashboard, Dispatch, Client Web...)

> **🔐 Security tools — install Day 1**
> Drata monitors your device for security compliance (encryption, antivirus, OS updates). 1Password is where the team stores SHARED credentials (API tokens, service accounts). Both use the Jitsu Google account for SSO login. If access is not granted yet, contact DevOps.

> **⚠️ StrongDM is the LONGEST access to provision**
> Request StrongDM + DB resources at least 5–7 days BEFORE your start date. DevOps approval typically takes 1–2 business days, and you'll need it from Day 2 onwards to query the database. Don't wait until Day 1 to request.

### 🛠 Tools to Install

- ☐ **Drata Agent** (installed on laptop, monitors device compliance — required for SOC 2)
- ☐ **1Password Desktop App** (+ browser extension for autofill)
- ☐ **StrongDM Desktop Client** (from app.strongdm.com/app/download) + optional sdm CLI
- ☐ Postman (latest) + import Jitsu API collection
- ☐ VS Code or your code editor of choice
- ☐ MongoDB Compass or Studio 3T (for browsing collections)
- ☐ DBeaver or pgAdmin (for PostgreSQL)
- ☐ Chrome/Firefox with bookmarks ready
- ☐ Mobile devices for testing: iOS (TestFlight) + Android
- ☐ Firebase App Distribution access (for downloading Staging builds)

### 📚 Pre-reading (optional but recommended)

- Skim the Jitsu QA Training Document — main reading material for Week 1
- Browse https://docs.gojitsu.com/ — public API documentation
- Read 1–2 closed Jira tickets from the QA team to see ticket format

---

# WEEK 1 — Foundation & Domain

*Goal: Understand the business, master the tools, run a full E2E flow.*

### Week 1 Outcomes

By Friday, you should be able to:

- Explain Jitsu's business model in 2 minutes to a non-engineer.
- Name all 4 driver types, 4 client service types, 4 warehouse types — and how they interact.
- Navigate fluently between Dashboard, Dispatch, Client Web, Recipient Web, DSP Portal.
- Run a complete E2E shipment flow on Staging by yourself.
- Read & understand a Jira ticket's acceptance criteria.

### Week 1 Milestone Check-in

*End of Day 5: 30-minute meeting with QA lead to demo your E2E run and ask blocking questions.*

## DAY 1 — Welcome & Big Picture

### 🌅 Morning — Orientation & Security Setup

- ☐ Meet QA lead + assigned buddy (30 min)
- ☐ Verify all accounts work (login to Jira, Confluence, QMetry, Slack)

#### 🔐 Security Tools Setup (required by IT for SOC 2 compliance)

- ☐ **Drata:** login at drata.com with Jitsu Google account — verify device shows up as monitored
- ☐ **Drata Agent:** install the local agent on your laptop (link in Drata dashboard) — runs in background, no daily action needed
- ☐ **1Password:** login at 1password.com with Jitsu Google account
- ☐ **1Password Desktop App:** install + sign in + browser extension
- ☐ Explore shared 1Password vaults — find which vaults the QA team has access to (ask QA lead/buddy)

> **⚠️ If Drata or 1Password access fails**
> These are critical for SOC 2 compliance and team credential sharing. Contact DevOps immediately if login doesn't work — don't proceed with other tasks until resolved.

- ☐ Read Training Document: Section 1 (About Jitsu) + Section 2 (Key Terminology)
- ☐ Watch any recorded company overview / all-hands intro (if available)

### 🛠 Afternoon — System Tour & StrongDM Setup

- ☐ Bookmark all 14 Staging URLs from Training Doc Section 6
- ☐ Log into Dashboard, Dispatch, Client Web, Recipient Web, DSP Portal — just click around
- ☐ Read Training Document: Section 6 (System Overview)

#### 🔐 StrongDM Setup (required before Day 2)

- ☐ **Read Training Doc Section 11.0** (Database & Tools Access via StrongDM)
- ☐ Login to app.strongdm.com with your Jitsu credentials
- ☐ Download & install StrongDM Desktop Client (+ sdm CLI if comfortable with terminal)
- ☐ Connect to PostgreSQL Staging resource — wait for ♥ Connected status
- ☐ Connect to MongoDB Staging resource — wait for ♥ Connected status
- ☐ Open DBeaver (or pgAdmin) → connect with host=localhost + port from SDM → verify can browse tables
- ☐ Open MongoDB Compass (or Studio 3T) → connect with host=localhost + port from SDM → verify can browse collections
- ☐ (Optional) Setup browser proxy for `*.gojitsu.sdm.network` — needed for Jenkins later

> **⚠️ If StrongDM access not yet approved**
> If your StrongDM account or DB resources aren't approved yet, DM your QA Lead immediately. You'll need DB access starting Day 2 for warehouse/region queries. As a fallback, ask a teammate to run queries for you during Days 2–3.

- ☐ Note 5 things in the UI you don't understand — bring to QA lead tomorrow

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 1 done — first overview of Jitsu systems and terminology'. Include what you learned, any blockers, and tomorrow's plan.

> **💡 DAY 1 TIP**
> Don't try to memorize everything. Focus on getting comfortable with the volume of new terms. You'll see them 100+ times in the next two weeks.

## DAY 2 — Warehouses, Regions & Zones

### 🌅 Morning — Geography of Jitsu

- ☐ Read Training Document: Section 3 (Warehouses, Regions & Zones)
- ☐ Memorize the 4 warehouse types: Main WH / Mobile Hub / Client WH / DSP WH
- ☐ Understand the hierarchy: USA → States → Regions → Zones
- ☐ **Internalize the rule: 1 Region = 1 Client Service Type**

### 🛠 Afternoon — Explore Real Data

- ☐ Open Dashboard → Region tab. List all regions you see. Note their client types.
- ☐ Open Dashboard → Warehouse tab. Identify Main WH vs Mobile Hub examples.
- ☐ In PostgreSQL (connected via StrongDM), query the `client_service` table — see how clients link to regions
- ☐ Open Dispatch → look at a route on the map. Identify which zone it belongs to.

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 2 done — Warehouses, Regions & Zones'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 3 — Client Service Types & Delivery Flows

### 🌅 Morning — The 4 Client Types

- ☐ Read Training Document: Section 4 (Client Service Types)
- ☐ Memorize enum codes: **NEXT_DAYS(0), ON_DEMAND(1), SPECIALTY(2), FCTD(3)**
- ☐ Understand each delivery flow — draw the flow diagrams on paper
- ☐ Distinguish: when does flow go through Inbound Sort vs direct from Client WH?

### 🛠 Afternoon — Live Examples

- ☐ In Dashboard → Client tab, find 1 client of each type (NEXT_DAYS, ON_DEMAND, SPECIALTY, FCTD)
- ☐ In Client web (Staging), look at a Commingle client's shipment list
- ☐ Query database: `SELECT logistic_type FROM assignments LIMIT 20` — see distribution
- ☐ Talk to QA lead: ask which clients in Prod are the most active per type

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 3 done — 4 Client Service Types and their flows'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 4 — Driver Types & Booking Methods

### 🌅 Morning — The 4 Driver Types

- ☐ Read Training Document: Section 5 (Driver Types & How They Get Routes)
- ☐ Memorize: IC / DSP / 3P / Linehaul — and 'Courier' = DSP + 3P only
- ☐ Master the Permission Matrix (Section 5.3) — who can book what
- ☐ Note Linehaul is a separate domain — don't go deep yet

### 🛠 Afternoon — Booking Methods Deep Dive

- ☐ Watch demo: Create Direct Booking from Booking Session (link in Section 11.6)
- ☐ In Dashboard, open the Forecast tab → look at a Booking Session
- ☐ In Dispatch → Schedule, look at a Direct Booking schedule
- ☐ Talk to a driver coordinator (if available) to hear the real ops perspective

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 4 done — Driver types and Booking permission matrix'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 5 — First E2E Run — Setup & Submit

> **🚀 BIG DAY**
> Today you'll run your first end-to-end flow on Staging. Don't worry about being slow — focus on understanding each step. By Friday next week, you'll do this in 30 minutes.

### 🌅 Morning — E2E Part 1: Client + Shipment

- ☐ Read Training Document: Section 11.1 + 11.2
- ☐ Create a test client on Staging Dashboard (prefix `Test_<YourName>`)
- ☐ Add `client_service` entries in PostgreSQL for 1–2 regions
- ☐ Pick a sample CSV from Project knowledge matching your client's region:
  - **Sample_CHI_Chicago.csv** — 39 shipments, Chicago region
  - **Sample_LAX_LosAngeles.csv** — 25 shipments, Los Angeles region
  - **Sample_SFO_SanFrancisco.csv** — 43 shipments, San Francisco region
  - **Sample_JFK_NewYork.csv** — 1,671 shipments, New York region
- ☐ Upload chosen CSV via Client web → Finalize batch
- ☐ Verify shipments appear in Dispatch

> **💡 TIP — Choosing the right sample CSV**
> For your first E2E run, pick Sample_CHI or Sample_LAX (smaller batches, faster to test end-to-end). Use Sample_JFK (1,671 shipments) only when you want to test multi-vehicle routing with a large volume.

### 🛠 Afternoon — E2E Part 2: Clone from Production

- ☐ Read Section 11.3 (Clone Shipment from Production)
- ☐ Use Postman 'Clone shipment for other client' API
- ☐ Download CSV → upload to Staging Client web for your test client
- ☐ Confirm cloned shipments are geocoded correctly

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 5 done — First E2E setup: test client created + shipments submitted'. Include what you learned, any blockers, and tomorrow's plan.

> **📅 END-OF-WEEK CHECK-IN**
> Schedule 30 minutes with your QA lead today or tomorrow. Demo what you set up. Discuss: 1) what was confusing this week, 2) what you want clarified before Week 2, 3) whether you're ready for the full E2E run on Monday.

---

## Weekend (Days 6–7) — Optional Reinforcement

*If you want to get ahead — or if Week 1 felt rushed — use the weekend lightly:*

### Light Reinforcement (1–2 hours total, not full days)

- Re-read any Training Document section that felt unclear.
- Browse 3–5 closed Jira tickets to see real ticket format + verification comments.
- Watch the Booking Session training demo if you haven't.
- Update your personal glossary with new terms from Week 1.

> **🛌 IMPORTANT**
> Don't work the full weekend. Burnout in Week 1 destroys the whole onboarding. Rest is part of the plan.

---

# WEEK 2 — Hands-on QA Work

*Goal: Run full E2E independently, write test cases, take real tickets.*

### Week 2 Outcomes

By Friday Day 14, you should be able to:

- Complete the full E2E flow on Staging independently in under an hour.
- Write test cases using the Jitsu QMetry template + import into QMetry.
- Pick up a real maintain ticket from Jira and produce a verification comment.
- Test something on mobile (Driver App from Firebase or TestFlight).
- Report progress in the QA channel daily without prompting.

### Week 2 Milestone Check-in

*End of Day 14: 30–45 minute meeting with QA lead. You demo a completed real ticket — including test cases in QMetry and Jira comment. Decision: ready to fully onboard onto sprint.*

## DAY 8 — Full E2E Run — Route, Driver, Deliver

### 🌅 Morning — Routing + Driver Setup

- ☐ Read Section 11.4 (Routing Web App) + 11.5 (Sign Up Driver)
- ☐ Take your client's shipments from Friday → create a Routing Problem
- ☐ Set up regions, vehicles, run 'Reroute All', select a solution
- ☐ Sign up a test driver via Driver App
- ☐ Use Postman 'Simulate fcra approval' to auto-approve

### 🛠 Afternoon — Booking + Delivery

- ☐ Create an Advance Booking Session (Section 11.6) OR a Schedule (11.7)
- ☐ Book a route as the test IC driver
- ☐ Use Postman 'Scan all shipments' (Section 11.9) to simulate inbound
- ☐ In Driver App, simulate delivery (pickup → arrive → POD photo → complete)
- ☐ Verify shipment status in Dispatch + Recipient web (tracking page)

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 8 done — Full E2E run completed (Route → Driver → Delivery)'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 9 — QA Process Deep Dive — Test Cases & QMetry

### 🌅 Morning — QA Process

- ☐ Read Training Document: Section 8 (QA Process & Workflow)
- ☐ Internalize the GOLDEN RULE for Maintain Tickets: check Prod flow + real data + End User viewpoint
- ☐ Read Section 9 (Test Case Template & QMetry)
- ☐ Open the official TCs Template Google Sheet — explore tabs

### 🛠 Afternoon — QMetry Practice

- ☐ Log into QMetry, navigate Test Case tab
- ☐ Find an existing folder (e.g. ENG-xxxx) to see real test case format
- ☐ Pick a simple Jira ticket (with buddy's help) — write 3–5 test cases using the template
- ☐ Import test cases to QMetry under a folder named with the ticket ID
- ☐ Create a Test Cycle following the naming convention: `[Staging]_[Ticket_ID]_Ticket name`

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 9 done — First test cases written and imported to QMetry'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 10 — First Real Maintain Ticket

> **🎯 BIG MILESTONE**
> Today you take your first real ticket from the backlog. QA lead will pick a small maintain ticket (low risk, well-scoped). Take your time — quality > speed.

### 🌅 Morning — Ticket Analysis

- ☐ QA lead assigns you a maintain ticket on Jira
- ☐ Read user story + AC carefully — write a 3-sentence summary in your own words
- ☐ Apply the GOLDEN RULE:
  - ☐ → Check current flow on Production
  - ☐ → Look at REAL data — how do end users actually behave?
  - ☐ → Build test cases from the End User viewpoint

### 🛠 Afternoon — Write & Run Test Cases

- ☐ Write 5–10 test cases covering happy path + edge cases
- ☐ Import to QMetry following the naming convention
- ☐ Create Test Cycle for Staging
- ☐ Start executing tests on Staging
- ☐ Watch Datadog logs while testing — ensure no errors after deploy
- ☐ If you find a bug while testing → log it on Jira using the Bug Reporting template (see Training Doc Section 8 — Step 8). Remember: set QA Review = yourself since you're the ticket owner.

> **🐛 First time logging a bug?**
> Day 10 is often when new QA members log their first real bug. Open the Training Document → Section 8 → Step 8 (Bug Reporting) for the full template, required fields, and real examples (MOB-2676 for mobile, ENA-1131 for web). Don't forget to link the bug back to your parent ticket via 'Linked Issues'.

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 10 done — First real maintain ticket picked up, test cases in progress'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 11 — Complete First Ticket — Verification

### 🌅 Morning — Finish Staging, Move to Beta

- ☐ Complete all test cases on Staging
- ☐ If any FAIL — write a detailed bug report on Jira, attach evidence
- ☐ If all PASS on Staging — verify config in Beta matches Prod, then test on Beta
- ☐ Watch Datadog throughout

### 🛠 Afternoon — Verification Comment

- ☐ Write a verification comment on the Jira ticket using the standard format:

```
Tested on Staging — PASSED

Test cases: [QMetry folder link]

Scenarios verified:
1. [Scenario 1]
2. [Scenario 2]
...

Evidence:
[Attach screenshots / videos of main flow]
```

- ☐ Attach evidence (screenshots, screen recordings of main flow)
- ☐ Tag QA lead in comment for review
- ☐ Move ticket to 'Ready to Release' or appropriate status

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 11 done — First ticket verified and Jira comment posted'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 12 — Mobile Testing Introduction

### 🌅 Morning — Mobile App Tour

- ☐ Read Training Document: Section 6.4 (Mobile Apps)
- ☐ Install Driver App from Firebase (Staging) on your test device
- ☐ Read Section 11.13 (Vehicle Insurance/Registration update flow)
- ☐ Explore Driver App: registration, profile, vehicle info, booking screen

### 🛠 Afternoon — Mobile Hands-on

- ☐ Reuse your test driver from Day 8 — log in on Driver App Staging build
- ☐ Book a route, do a simulated pickup/delivery flow
- ☐ Take a POD photo (test the geofence behavior)
- ☐ Test 1 small mobile feature from the current sprint — ask buddy for a candidate
- ☐ Understand the mobile workflow: Staging Firebase → all pass → TestFlight → Prod

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 12 done — Mobile testing introduction on Driver App Staging'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 13 — Production Testing & Safety

> **🚨 CRITICAL DAY**
> Today you learn the Production rules. Re-read them every time you touch Prod. Mistakes in Prod affect real merchants, drivers, and revenue.

### 🌅 Morning — The 8 Production Rules

- ☐ Read Training Document: Section 10 (Important Rules for QA in Production)
- ☐ Memorize the 8 rules — quiz yourself
- ☐ Talk to QA lead about specific Prod incidents to learn from

### 🛠 Afternoon — Safe Prod Practice

- ☐ With QA lead supervision, log into Prod Dashboard
- ☐ Practice the 'test client' workflow on Prod (prefix `Test_`)
- ☐ Practice cleanup — delete a test assignment/BS after use
- ☐ Verify your Jitsu email is being used (no personal email!)
- ☐ Set up a 'Prod testing checklist' you'll use every time before touching Prod

### ✅ Daily Slack Report

Post in the QA channel using the standard format: 'Day 13 done — Production rules learned + Prod safety checklist ready'. Include what you learned, any blockers, and tomorrow's plan.

## DAY 14 — Wrap-up & Sprint Onboarding

### 🌅 Morning — Knowledge Consolidation

- ☐ Take a second small ticket — execute end-to-end independently (no handholding)
- ☐ Write test cases, run on Staging + Beta, post verification comment
- ☐ Aim to finish in 3–4 hours (you've done this before)

### 🛠 Afternoon — Final Milestone Meeting

- ☐ 30–45 minute meeting with QA lead
- ☐ Demo: Walk through your second ticket from analysis → verification comment
- ☐ Discuss: What clicked? What's still confusing?
- ☐ Decision: Are you ready to be assigned regular sprint tickets?

### ✅ Daily Slack Report

- Post your final Day 14 report in the QA channel: 'Day 14 done — Second ticket completed independently. Onboarding wrap-up!'
- Optionally: share your 'Lessons Learned' reflections (1 paragraph in Slack is enough — no need for a separate doc).

> **🎉 ONBOARDING COMPLETE**
> By the end of Day 14, you should feel ready to take regular sprint tickets. You won't be an expert yet — that takes months — but you'll have the foundation to learn fast.

---

# Appendix A — Skills Checklist

Use this for a quick self-assessment. By end of Day 14, you should confidently tick most boxes.

### Domain Knowledge

- ☐ Can explain Jitsu's business model in 2 minutes
- ☐ Know all 4 warehouse types and their roles
- ☐ Know the Region → Zone hierarchy and the '1 Region = 1 Client Type' rule
- ☐ Know all 4 client service types (enum codes + flows)
- ☐ Know all 4 driver types + the Permission Matrix
- ☐ Can define: Shipment, Route, Assignment, POD, OTD, SLA, Stem miles, Sprinkling, Geofencing

### System Navigation

- ☐ Can navigate Dashboard, Dispatch, Routing, Client Web, Recipient Web, DSP Portal
- ☐ Know the URL pattern for Staging/Beta/Prod
- ☐ Have bookmarked all 14 Staging URLs
- ☐ Can use Inbound, Small Sort, Payment, Geocoder, Admin tools
- ☐ Can install + use Driver App from Firebase (Staging)

### Tools

- ☐ **Drata:** agent installed, device shows compliant in dashboard
- ☐ **1Password:** can access shared vaults, use browser extension for autofill
- ☐ **StrongDM:** can connect to DB resources, configure browser proxy for internal apps
- ☐ Postman: can run requests, use environments, chain requests
- ☐ MongoDB Compass: can query collections (sort_center, driver_registration_record, etc.)
- ☐ PostgreSQL: can query client_service, assignments tables
- ☐ Datadog: can find logs for a deploy or service
- ☐ QMetry: can create folders, import test cases, run test cycles
- ☐ Jira: can move tickets, write verification comments

### E2E Flow

- ☐ Can create a test client (Dashboard + client_service table)
- ☐ Can submit shipments (CSV batch + single API)
- ☐ Can clone shipments from Production
- ☐ Can create a Routing Problem and select a solution
- ☐ Can sign up a driver + simulate FCRA approval
- ☐ Can create a Booking Session or Schedule
- ☐ Can scan shipments automatically via Postman
- ☐ Can trigger weekly payment via the worker

### QA Process

- ☐ Can write test cases using the Jitsu template
- ☐ Can import test cases to QMetry with proper folder structure
- ☐ Can create Test Cycles with the correct naming convention
- ☐ Can write a verification comment that includes env, result, scenarios, evidence
- ☐ Can log a bug ticket following the template (10 required fields + Mobile-specific Fix Version)
- ☐ Know the QA Review field rule: yourself if from a ticket you're assigned, optional if standalone
- ☐ Know all 8 Production rules
- ☐ Apply the GOLDEN RULE: check Prod flow + real data + End User viewpoint

---

# Appendix B — Common Pitfalls

Things that trip up new QAs at Jitsu. Forewarned is forearmed.

### 🚫 Pitfall 1: Calling IC drivers 'couriers'

In Jitsu, 'courier' is reserved for DSP + 3P drivers only. IC = Independent Contractor, never courier. Using the wrong term in a ticket comment will confuse the team.

### 🚫 Pitfall 2: Confusing 'Route' and 'Assignment'

Route = the optimized path. Assignment = shipments + route, ready to be booked/assigned. They're related but distinct. Read tickets carefully.

### 🚫 Pitfall 3: Forgetting '1 Region = 1 Client Type'

You'll try to assign a client to a region that doesn't match its type, and wonder why it fails. Always check the region's type first.

### 🚫 Pitfall 4: Using personal email in Prod

Always Jitsu email (@jitsu / @gojitsu / @axlehire). Personal emails get rejected and create messy data.

### 🚫 Pitfall 5: Forgetting to clean up Prod test data

You created a test assignment / Booking Session / Schedule on Prod for testing. Delete it after. Old test data clutters real ops.

### 🚫 Pitfall 6: Skipping the 'check Prod flow first' step for maintain tickets

Easy to skip when you're busy. But maintain tickets WITHOUT real-data context lead to incomplete test cases. Always check Prod first.

### 🚫 Pitfall 7: Assuming Beta = Production

Beta is pre-prod, but config might drift. Always verify Beta config matches Prod before drawing test conclusions.

### 🚫 Pitfall 8: Not watching Datadog while testing

You see the UI work, but a backend error happens silently. Datadog catches what the UI doesn't. Watch logs after every deploy.

---

# Appendix C — Who to Ask

When stuck, ask in this order — don't burn an hour Googling. Default to asking after 15–20 minutes stuck.

| Question type | Who / Where |
| --- | --- |
| **How does X work in product?** | 1. Confluence search → 2. QA buddy → 3. QA lead |
| **Where's the documentation for X?** | Training Document first → Confluence ENG space → ask in QA channel |
| **Why does the code do X?** | Dev assigned to ticket → ticket author → engineering channel |
| **How do I use QMetry/Jira?** | QA buddy → QA lead → search QMetry docs |
| **Production behavior / real data?** | QA lead supervises Prod access → Client Success for ops context |
| **Mobile app build issues?** | qa-mobile channel → mobile dev lead |
| **Account / access / permissions?** | IT / DevOps Slack channel → QA lead can escalate |
| **Drata / 1Password / StrongDM issues?** | DevOps channel directly (security & infra tools) |
| **Anything urgent / blocking?** | DM QA lead directly — don't wait |

> **💡 ASKING TIP**
> When you ask, share: 1) what you're trying to do, 2) what you tried, 3) what happened. Don't just say 'this doesn't work'. Good questions get fast answers.

---

# Appendix D — Internal Contacts

Key people across Jitsu departments. Use this as your reference when escalating issues or coordinating cross-team work.

*Source: [Contacts — Confluence](https://gojitsu.atlassian.net/wiki/spaces/DT/pages/1093173313/Contacts)*

> **📌 Keep this up-to-date**
> Team structure changes over time. Always cross-check with the Confluence Contacts page (linked above) for the most current info. Personal phone numbers in this doc may not be up to date.

## Key Departments

| Department / Role | Point of Contact | Notes |
| --- | --- | --- |
| **CTO** | Evan Robinson | Top-level technical decisions |
| **Head of Engineering** | Dzung Le Quang | Engineering escalations |
| **IT Service** | b.gombodash@gojitsu.com | Daily requests: post to #it on Slack or email it@gojitsu.com |
| **Human Resources (VN)** | Manager: ha.do@gojitsu.com; Member: quynh.dinh@gojitsu.com | Onboarding paperwork, leave, benefits |
| **DevOps** | Manager: Mo Kassem; Members: Dat Tran, Hoang Do | StrongDM, infrastructure, deploy issues |
| **Product** | Head of Product: Chantra Park; Scrum Master: Phuong Le | Product roadmap, sprint ceremonies |

## Product Managers / Product Owners

| Person | Scope |
| --- | --- |
| **Sampson Wu** | PM of Engineering's INB (Inbound/Warehouse) and ENA teams |
| **Mi Nguyen** | PO of Engineering's WAT (Web App) and MOB (Mobile) teams |
| **Huyen Nguyen** | PO of Engineering's Routing team |

## Engineering Team Leads

When you need technical context for a ticket, the team lead is your best escalation point after the assigned developer.

| Team | Lead | Domain (for QA context) |
| --- | --- | --- |
| **Web App Team (WAT)** | Linh Lam | Dashboard, Dispatch, Client/Recipient/DSP web |
| **Inbound/Warehouse Team (INB)** | Huy Tran | Inbound, Small Sort, Warehouse App, dimensions |
| **Routing Team** | Cuong Dang | Routing engine, assignments, optimization |
| **Mobile Team (MOB)** | Bach Mai | Driver App, Outbound App, Warehouse App (mobile) |
| **Automation QA** | Diu Vu | Test automation framework, CI integration |
| **Manual QA** | Hue Chu | Manual testing, QA process, your direct chain of command |

> **💡 For QA — when to escalate to which team**
> Ticket on web feature → WAT (Linh Lam). Routing issue → Routing (Cuong Dang). Mobile app bug → MOB (Bach Mai). Inbound/Warehouse flow → INB (Huy Tran). Manual QA process question → Hue Chu (your direct lead). Always copy your QA lead on cross-team escalations.

---

**End of Onboarding Plan — You've got this! 🚀**

*Refer to the Jitsu QA Training Document throughout these 14 days.*
