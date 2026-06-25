# Jitsu QA — Probation Goals

> **Version 1.2** • June 13, 2026
> Your goals for the 2-month probation at Jitsu — Junior QA & Senior QA

---

## 📋 Document Version History

| Version | Date | Changes |
|---------|------|---------|
| **v1.2** | 2026-06-13 | Added Section 9 — "Future of QA at Jitsu" — explaining career direction (Mar 2026 transition to QI/QS disciplines, 9-18 months). New members aware that Manual QA path → QI Engineering. References to Career Framework doc. No threshold or category changes. |
| **v1.1** | 2026-05-26 | Added Feature Documentation skills to both levels (Category 3): Junior — Feature Documentation Hygiene + Feature Flag Documentation. Senior — Feature Documentation Ownership + Feature Flag Documentation Standards. Added a "Documentation discipline" section to Tips (Section 8) with concrete examples (`claimsProcessingEnabled`, `SLA_VERSIONING_ENABLED`, `enable_require_id_scan`). |
| v1.0 | 2026-05-26 | Initial pilot release. Junior QA (5 goal areas, 33 skills) + Senior QA (7 goal areas, 41 skills). 2 outcomes (PASS / NOT PASS). Includes: Deep domain knowledge (Shipment + Assignment Lifecycle full flow, Cross-team Knowledge), AI fluency (4 skills per level — usage, prompt engineering, verification, workflow design), Business + Product mindset, Risk-based thinking (Senior), Daily progress reporting (non-negotiable Day 1). |

> **📌 Versioning convention**
> - **Minor updates** (e.g. `v1.0 → v1.1`) — wording tweaks, single skill adjustment, clarification.
> - **Major updates** (e.g. `v1.x → v2.0`) — adding/removing categories, changing thresholds, restructuring goal areas.
>
> If you have an older version, check this changelog to see what's new before discussing with your QA Lead.

---

## Table of Contents

1. Welcome & Your Probation Timeline
2. Junior QA — Your 5 Goal Areas
3. Senior QA — Your 7 Goal Areas
4. How You'll Be Assessed
5. Evidence That Counts
6. Pass / Not Pass Thresholds
7. What Happens at Each Outcome
8. Tips to Succeed During Probation

---

## 1. Welcome & Your Probation Timeline

Welcome to the Jitsu QA Team! This document outlines what you should achieve during your 2-month (60-day) probation period. Read it on Day 1 — it tells you exactly what "good" looks like at the end of probation, so you can set your own goals from the start.

### Why this document exists
- You know upfront what's expected — no surprises at Day 60.
- You can self-assess your progress along the way and ask for help where needed.
- Your QA Lead uses the same criteria — fair and consistent evaluation.
- Helps you plan your learning + identify gaps early.

### Your Probation Timeline

| Milestone | When | What happens |
|-----------|------|--------------|
| **Day 1** | Your first day | Onboarding kickoff. Your QA Lead walks you through this document. You receive the Welcome Kit + 2-Week Onboarding Plan. Start setting your own goals. |
| **Day 14** | End of week 2 | Informal check-in with QA Lead. Review onboarding progress. Raise any blockers — don't wait until they pile up. |
| **Day 30** | Mid-probation | Formal 1:1 review. You self-assess + QA Lead assesses. Compare in 1:1. This is a chance to course-correct, not a final judgment. |
| **Day 45** | Pre-final check | Light check-in. Confirm you're on track. If gaps exist, agree on focus areas for the final 2 weeks. |
| **Day 60** | Final evaluation | Final review. QA Lead determines outcome: Pass / Not Pass. Discussion with Engineering Lead before submitting to HR. |

> **💡 Probation is a 2-way relationship**
> While we're evaluating whether you're a good fit for Jitsu, you're also evaluating whether Jitsu is a good fit for you. Don't hesitate to raise questions about: the role, the team, what you need to succeed, what feels confusing. We'd rather you ask early than struggle silently.

> **📢 Daily progress reports — non-negotiable, starting Day 1**
> Post a brief daily update in the QA channel every working day. Format: what's done today, what blockers, what's planned tomorrow. This is how your QA Lead stays informed without having to chase you — and how the team identifies issues early.
>
> If you hit an issue (technical blocker, ticket unclear, environment broken, anything affecting your work) — **DON'T** wait for daily standup or your next 1:1. Ping your Lead immediately. Early visibility = faster resolution.
>
> Missing daily reports or hiding blockers is the single most common reason a probation goes sideways. Don't make that mistake.

---

## 2. Junior QA — Your 5 Goal Areas

As a Junior QA, here's what you should achieve by Day 60. You're expected to learn the Jitsu domain rapidly, follow the QA process, and contribute to sprint with appropriate scope tickets. Don't underestimate the domain — Jitsu's logistics + delivery flow has real depth, and you'll be expected to understand it deeply (not just superficially) by end of probation.

### Weight distribution

| # | Category | Weight | Focus |
|---|----------|--------|-------|
| 1 | Domain Knowledge | 20% | Deep understanding of Jitsu domain + cross-team |
| 2 | Basic Technical Skills | 20% | E2E + Postman + DB + Datadog + Mobile + AI fluency (prompts, verification) |
| 3 | QA Process | 25% | Test cases, QMetry, bug reporting, release process, doc hygiene |
| 4 | Quality of Work | 20% | Completion + thoroughness + learning velocity |
| 5 | Soft Skills & Attitude | 15% | Communication + business mindset + curiosity |
| **TOTAL** | | **100%** | |

### Category 1 — Domain Knowledge (20%)

The Junior QA must build deep understanding of Jitsu's business and entities during probation. The domain is complex — but mastering it is essential for everything else.

| Skill | Expectation |
|-------|-------------|
| **Jitsu Business Model** | Understands what Jitsu does (last-mile delivery) + 3 parties (Client / Driver / Recipient) and how they interact. |
| **Client Service Types** | Understands the 4 client service types in the `service_type` table (`COMMINGLE` / `ONDEMAND` / `SPECIALTY` / `FCTD`) and their delivery flows. Knows they map to the `logistic_type` enum on `assignments` after routing (`NEXT_DAYS=0` / `ON_DEMAND=1` / `SPECIALTY=2` / `FCTD=3`, where `COMMINGLE → NEXT_DAYS`). Knows the difference between commingle and direct flows. |
| **Driver Types (Deep)** | Knows IC / DSP / 3P / Linehaul + their booking permission matrix (only IC can self-book; DSP/3P assigned directly). Understands edge cases: Linehaul auto-assignment, sprinkling. Doesn't confuse 'courier' (= DSP + 3P) with IC. |
| **Warehouses & Regions** | Knows 4 warehouse types + when each is used. Understands the region/zone hierarchy + the rule: 1 Region = 1 Client Type. |
| **Shipment Lifecycle (Full Flow)** | Understands full status flow: `DRAFT/CREATED → GEOCODED → ASSIGNED → PICKUP_* → DROPOFF_* / RETURN_* / CANCELLED_*`. Knows what each failure status means (`PICKUP_FAILED`, `DROPOFF_FAILED`, `RESCHEDULED`, `RETURN_TO_CLIENT`). |
| **Assignment Lifecycle (Full Flow)** | Understands assignment status flow + stop statuses (`PENDING` / `EN_ROUTE` / `SUCCEEDED` / `FAILED`). Knows sprinkling (adding shipment to existing assignment) + dispatcher rescue scenarios. |
| **Cross-team Knowledge** | Understands how WAT (Web App) / INB (Inbound/Warehouse) / Routing / MOB (Mobile) teams interact. Knows which team owns which feature. |
| **Core Terminology** | Comfortable with: Shipment / Route / Assignment / POD / OTD / SLA / Geofencing / Sprinkling. |

### Category 2 — Basic Technical Skills (20%)

Junior is expected to USE QA tooling, not master it. Help from buddy / doc is acceptable. Strong AI usage is a force multiplier — leverage it.

| Skill | Expectation |
|-------|-------------|
| **E2E Flow Execution** | Can run E2E flow on Staging WITH GUIDANCE from buddy or doc. Doesn't need to do it alone yet. |
| **Postman Basics** | Can use the QA Master Collection for simple requests. Refreshes tokens correctly. |
| **Database — Read Only** | Can connect via StrongDM + run basic SELECT queries from the Cheat Sheet. |
| **Datadog — Find Logs** | Can find logs in Datadog for a specific service when shown how. Understands what logs to look at. |
| **Mobile App Testing** | Tests Driver App across iOS + Android devices. Familiar with Firebase (Staging build distribution) + TestFlight (Prod build distribution) workflow. |
| **AI Tools for Productivity** | Uses AI tools (e.g. Claude, ChatGPT, Augment) for daily work — generating test ideas, drafting test cases, summarizing tickets, writing bug reports, explaining unfamiliar code. |
| **Prompt Engineering (Basic)** | Writes effective prompts: provides context (ticket info, code snippets, AC), specifies output format, breaks down complex tasks. Iterates on prompts when first output is poor — doesn't give up after one try. |
| **AI Verification & Critical Thinking** | Recognizes AI hallucinations (fake API names, wrong status codes, made-up function signatures). Always cross-checks AI output against actual codebase / docs / Confluence before acting on it. Treats AI as an assistant, not authority. |
| **AI-assisted Domain Research** | Uses AI to navigate Jitsu's complex domain efficiently — summarizing long Confluence pages, finding relevant tickets, understanding unfamiliar code areas. Combines AI with Confluence search to onboard faster. |

### Category 3 — QA Process (25%)

The highest-weight category. Junior must follow Jitsu's QA process — accuracy more important than speed.

| Skill | Expectation |
|-------|-------------|
| **Test Case Writing** | Writes basic test cases using the Jitsu template. Covers happy path. May miss edge cases — that's OK at this level. |
| **QMetry Usage** | Creates a folder per ticket. Imports test cases. Follows the naming convention. |
| **Verification Comments** | Posts a comment per env (Staging / Beta / Prod) with status + evidence. May need reminders on build version. |
| **Bug Reporting** | Logs bugs per template (10 required fields). Mobile bugs include Fix Version. Evidence is clear. |
| **Release Process Awareness** | Understands the isolated-release-branch concept at a high level. Knows the rule to smoke test previous tickets on the same repo. |
| **Feature Documentation Hygiene** | When working on a feature: searches Confluence for an existing feature doc first. If it exists → updates it to reflect new behavior. If not → asks Lead whether to write a new one (bonus if proactive). Doesn't let docs go stale. |
| **Feature Flag Documentation** | When a ticket introduces a new feature flag, writes a Confluence doc capturing: flag name, location (Consul / Firebase / `item_metadata` / etc.), default value, when to enable/disable, scope (region / client / global). References existing examples (e.g. `enable_require_id_scan`, `claimsProcessingEnabled`, `SLA_VERSIONING_ENABLED`). |
| **Production Safety** | Knows the 8 Production Rules. Always asks a supervisor before doing anything on Prod. |

### Category 4 — Quality of Work (20%)

Junior outputs measured by completion + thoroughness within their scope. Learning velocity matters.

| Skill | Expectation |
|-------|-------------|
| **Ticket Completion** | Completes assigned tickets within the sprint. Ticket scope adjusted to Junior level. |
| **Test Thoroughness** | Tests AC scenarios. Catches obvious bugs. May miss subtle ones — that's expected at Junior level. |
| **Evidence Quality** | Attaches screenshots / videos to verification comments. Evidence is visible and clear. |
| **Learning Velocity** | Picks up new concepts week by week. Asks questions to learn faster. Demonstrates growth Day 1 → Day 60. |

### Category 5 — Soft Skills & Attitude (15%)

For Junior, attitude and willingness to learn are CRITICAL. Skills can be taught — attitude is the foundation.

| Skill | Expectation |
|-------|-------------|
| **Daily Progress Reports to Lead** | Posts daily updates in the QA channel so the QA Lead always knows your status. Reports include: what's done today, what blockers, what's planned tomorrow. NEVER skip — if no progress, still post and explain why. |
| **Proactive Issue Escalation** | If you hit an issue (technical blocker, ticket unclear, environment broken, personal issue affecting work) — report it to Lead IMMEDIATELY. Don't wait for daily standup or until it becomes critical. The earlier Lead knows, the easier to unblock you. |
| **Asking Questions** | Uses the 15-min rule — asks for help after 15 min stuck. Not afraid to ask 'dumb' questions. |
| **Receptive to Feedback** | Takes coaching well. Applies feedback in the next ticket without defensiveness. |
| **Team Engagement** | Engages in team rituals (standup, retro, training). Shows interest in learning beyond just assigned work. |
| **Business Mindset (not Outsource Mindset)** | Doesn't just "complete what's in the ticket and stop". Thinks about the business goal behind the ticket — why does it matter to the user / client? Goes beyond the literal scope when needed. |
| **Product Curiosity** | Curious about the Jitsu product as a whole — not just their assigned area. Explores other features, asks about how things work, raises questions that show genuine interest. |
| **Raises Risks Proactively** | If they spot a potential risk or issue (even outside their ticket), they raise it. Not limited to "my ticket only" thinking. |

---

## 3. Senior QA — Your 7 Goal Areas

As a Senior QA, here's what you should achieve by Day 60. You're expected to handle complex tickets independently, mentor others, drive process improvements, and contribute strategically to quality at Jitsu.

### Weight distribution

| # | Category | Weight | Focus |
|---|----------|--------|-------|
| 1 | Domain Knowledge | 15% | Deep domain + cross-team mastery |
| 2 | Technical Skills | 15% | Advanced tooling + API + strategic AI usage + lead AI adoption |
| 3 | QA Process | 20% | Test strategy + process mastery + doc ownership |
| 4 | Quality of Work | 15% | High completion + low defect escape rate |
| 5 | Independence & Risk-based Thinking | 15% | Risk-based test approach + ownership |
| 6 | Mentoring, Knowledge Sharing & Communication | 10% | Daily reporting + junior support + documentation |
| 7 | Strategic Thinking & Business Mindset | 10% | Impact analysis + business mindset + product curiosity |
| **TOTAL** | | **100%** | |

### Category 1 — Domain Knowledge (15%)

Senior is expected to have deep domain knowledge AND guide juniors through it.

| Skill | Expectation |
|-------|-------------|
| **Jitsu Business Model** | Can explain to a non-engineer in 2 minutes. Knows current business priorities. Can mentor juniors on domain basics. |
| **Client Service Types** | Deep understanding of the `service_type` values (`COMMINGLE` / `ONDEMAND` / `SPECIALTY` / `FCTD`) and their mapping to the `logistic_type` enum on `assignments` after routing (`NEXT_DAYS=0` / `ON_DEMAND=1` / `SPECIALTY=2` / `FCTD=3`) + flows + Brand hierarchy (Parent / Brand / Brand-with-client_id). |
| **Driver Types (Deep)** | Knows IC / DSP / 3P / Linehaul + full booking permission matrix. Understands all edge cases (Linehaul auto-assignment, sprinkling logic, 3P escalation conditions). |
| **Warehouses & Regions** | Understands all 4 warehouse types + when each is used. Knows regional differences and operational nuances. |
| **Shipment Lifecycle (Full Flow)** | Knows full status flow: `DRAFT/CREATED → GEOCODED → ASSIGNED → PICKUP_* → DROPOFF_* / RETURN_* / CANCELLED_*`. Understands edge cases: `PICKUP_FAILED` recovery, `RESCHEDULED`, `RETURN_TO_CLIENT` flow, virtual statuses. |
| **Assignment Lifecycle (Full Flow)** | Knows assignment status flow + stop statuses (`PENDING` / `EN_ROUTE` / `SUCCEEDED` / `FAILED`). Understands sprinkling + dispatcher rescue + abnormal route scenarios. |
| **Cross-team Knowledge** | Understands how WAT / INB / Routing / MOB teams interact. Knows handoffs + cross-team dependencies. Knows when to escalate to which team. |

### Category 2 — Technical Skills (15%)

Senior must demonstrate advanced fluency in QA tooling AND lead the team on effective AI usage.

| Skill | Expectation |
|-------|-------------|
| **E2E Flow Mastery** | Runs full E2E in < 1 hour. Can guide a Junior through it without notes. |
| **Postman Advanced** | Writes pre-request scripts. Chains requests across multiple endpoints. |
| **Database Mastery** | Comfortable querying both PostgreSQL + MongoDB. Writes JOINs + aggregations. |
| **API Understanding** | Understands API contracts. Reviews API specs for testability concerns. |
| **Mobile App Testing** | Tests Driver App across iOS + Android devices. Familiar with Firebase (Staging build distribution) + TestFlight (Prod build distribution) workflow. Can guide juniors through the mobile testing process. |
| **AI Tools (Strategic Use)** | Uses AI (Claude, ChatGPT, Augment, Claude Code, DevOps GPT) strategically to boost productivity AND quality — generating comprehensive test cases, analyzing code changes for regression risk, reviewing test coverage, drafting clear bug reports, accelerating documentation. Picks the right AI tool for the right job. |
| **Prompt Engineering (Advanced)** | Writes precise, structured prompts that consistently get useful output the first try. Knows when to give AI more context vs less, how to break complex tasks into steps, how to constrain output format. Shares effective prompts + workflows with juniors. |
| **AI Verification & Hallucination Detection** | Spots AI hallucinations fast (fake APIs, wrong domain logic, made-up Jitsu terminology). Always verifies against codebase / Confluence — especially for business-critical decisions. Trains juniors to do the same. |
| **AI-assisted Workflow Design** | Designs efficient AI-assisted workflows for the team — which tasks to AI-assist, how to chain AI tools (e.g. Augment for code context + Claude for test strategy + DevOps GPT for SDLC). Stays current on new AI capabilities relevant to QA. |

### Category 3 — QA Process (20%)

Senior is expected to demonstrate mastery of the QA process and define the test approach.

| Skill | Expectation |
|-------|-------------|
| **Test Strategy** | Defines the test approach per ticket. Identifies risks early. Adjusts test scope based on impact. |
| **Test Case Quality** | Test cases cover happy + edge + negative + integration scenarios systematically. |
| **QMetry & Documentation** | Maintains a clean QMetry structure. Contributes to the test case library for re-use. |
| **Bug Reporting Excellence** | Bug reports clear, reproducible, with strong evidence. Includes impact analysis. |
| **Verification Discipline** | Comments on Jira after EACH env. Includes build version + scope + evidence consistently. |
| **Release Process Mastery** | Handles hotfix + partial release flows correctly. Knows when to skip Beta (e.g. no worker on Beta). |
| **Feature Documentation Ownership** | Owns feature documentation discipline. For every feature shipped: ensures a Confluence doc exists + is up to date. Writes new docs when none exist. Trains juniors to do the same. Treats stale docs as a quality issue, not just a nice-to-have. |
| **Feature Flag Documentation Standards** | Writes high-quality Confluence docs for every new feature flag: name, location (Consul / Firebase / `item_metadata`), default value, enable/disable conditions, scope (region / client / global), rollback plan, expected behavior when the flag flips. Reviews juniors' flag docs. |
| **Production Safety** | Follows the 8 Production Rules religiously. Trains juniors on Prod safety. |

### Category 4 — Quality of Work (15%)

Senior is expected to handle complex tickets with high accuracy.

| Skill | Expectation |
|-------|-------------|
| **Ticket Completion Rate** | Completes ≥ 90% assigned tickets on-time. Handles complex / multi-system tickets. |
| **Defect Escape Rate** | Bugs found AFTER QA verified < 10%. Solid coverage on Senior-level tickets. |
| **Test Coverage Depth** | Covers happy + edge + negative + performance considerations on complex tickets. |
| **Bug Triage** | Logs bugs with correct priority. Distinguishes regressions vs new issues correctly. |

### Category 5 — Independence & Risk-based Thinking (15%)

Senior QA must think in terms of risk + impact, not just tickets. Operates as a self-directed contributor.

| Skill | Expectation |
|-------|-------------|
| **Risk-based Test Approach** | Identifies HIGH / MEDIUM / LOW risk areas per ticket and allocates test effort accordingly. Doesn't test everything equally — focuses where impact is highest. |
| **Risk Identification** | Proactively flags risks to PM / PO BEFORE they become production bugs. Catches risks in planning / refinement, not after deploy. |
| **Process Improvement** | Suggests improvements to the QA process. Implements when approved. |
| **Decision Making** | Makes judgment calls on test scope and priority. Can justify decisions to the Engineering Lead with risk reasoning. |

### Category 6 — Mentoring, Knowledge Sharing & Communication (10%)

Senior is expected to lift the team and maintain transparency — not just deliver own tickets.

| Skill | Expectation |
|-------|-------------|
| **Daily Progress Reports** | Posts daily updates in the QA channel — even though no one asks. Lead always knows where the Senior's work stands without having to chase. Senior models the reporting culture for juniors. |
| **Proactive Issue Escalation** | When hitting an issue (blocker, risk, environment problem, ticket scope confusion), raises it to Lead immediately — not at the next 1:1. Includes proposed solutions, not just problems. |
| **Junior Support** | Helps junior QA when asked. Patient. Explains the 'why' behind decisions. |
| **Documentation Contribution** | Writes / updates internal docs. Captures team knowledge. |
| **Knowledge Sharing Sessions** | Presents learnings, demos new tools, shares tips in team meetings. |

### Category 7 — Strategic Thinking & Business Mindset (10%)

Senior contributes to quality decisions beyond just executing tests, with strong business + product awareness.

| Skill | Expectation |
|-------|-------------|
| **Impact Analysis** | Understands how a ticket impacts other features. Plans regression accordingly. |
| **Cross-team Communication** | Communicates effectively with Dev / PO / Eng Leads. Pushes back constructively when needed. |
| **Quality Advocacy** | Pushes for quality at planning / refinement stages — not just at the QA stage. |
| **Business Mindset (not Outsource Mindset)** | Doesn't just complete what's in the ticket. Thinks about the business goal — why does this feature matter to the client / driver / recipient? Suggests changes when ticket scope doesn't match the business goal. |
| **Product Curiosity** | Genuinely curious about the Jitsu product as a whole — not just their assigned area. Explores how features interact, asks 'why' often. Brings outside knowledge / industry awareness into discussions. |

---

## 4. How You'll Be Assessed

Each skill is scored on a 1–5 scale:

| Score | Label | Description |
|-------|-------|-------------|
| **5** | Excellent | Consistently exceeds expectations. Could mentor others. |
| **4** | Strong | Meets expectations consistently. Reliable. |
| **3** | Acceptable | Meets minimum expectations. Some inconsistency. |
| **2** | Needs Improvement | Below expectations. Requires coaching. |
| **1** | Unacceptable | Significant gap. Cannot perform this skill independently. |

### How the overall score is calculated

1. Each skill gets scored 1–5 based on observed behavior + concrete evidence.
2. Category average score = mean of all skills in that category.
3. Weighted % = `(category_avg / 5) × category_weight`.
4. Overall % = sum of all weighted % (out of 100).

> **💡 Score consistently, not generously**
> Default to score 3 (Acceptable) unless you have evidence to score higher or lower. Score 5 should be rare — it means truly exceptional. Score 1 is a serious concern. Most skills should land at 3 or 4 for a passing candidate.

---

## 5. Evidence That Counts

Your evaluation isn't based on impressions — it's based on concrete evidence. Knowing what counts helps you focus on the right things during probation.

### What evaluators look at

| Source | What it tells the evaluator |
|--------|----------------------------|
| **Your Self-assessment** | Your own view of your performance. Be honest — under-rating yourself is as unhelpful as over-rating. Submit it before each formal review. |
| **QA Lead's Observation** | Direct observation by your QA Lead throughout probation. They watch how you work, communicate, and grow. |
| **Cross-team Feedback** | Feedback from Devs, POs, PMs you worked with. Communication + collaboration are visible to them. |
| **Your Concrete Output** | Your actual tickets in Jira, test cases in QMetry, bug reports, verification comments, Slack reports. These are the artifacts that prove your skills. |

### What concrete metrics matter

- `#` of tickets you completed during probation.
- `#` of bugs you logged + their quality (clear repro steps, evidence).
- `#` of test cases you wrote in QMetry.
- `#` of bugs that escaped to Production after you verified — **key quality metric**.
- `#` of doc contributions or updates.
- Quality of your verification comments — is evidence always attached?
- Daily Slack reports consistency — did you post every working day?
- Specific incidents — positive (caught a critical bug) or negative (missed a regression).

> **📝 Build your own evidence file as you go**
>
> Don't wait until Day 60 to recall what you did. Keep a personal log:
> - "Ticket `WAT-2017` — wrote 8 test cases, found 2 bugs pre-deploy"
> - "Bug `MOB-2745` — well-reproduced, correct Fix Version set"
> - "Suggested improvement in retro — adopted by team"
>
> This helps you self-assess accurately AND gives concrete examples to discuss in 1:1s with your QA Lead.

---

## 6. Pass / Not Pass Thresholds

After all scores are calculated, the overall % score determines whether you've passed probation.

### Junior QA Thresholds

Junior bar is more forgiving — they're learning. Focus on growth trajectory + attitude.

| Outcome | Condition |
|---------|-----------|
| ✅ **PASS — confirm role** | Overall ≥ 65% AND every category ≥ 45% |
| ❌ **NOT PASS — not a fit** | Overall < 65% OR any category < 45% |

### Senior QA Thresholds

Senior bar is higher — they're expected to operate at a more advanced level.

| Outcome | Condition |
|---------|-----------|
| ✅ **PASS — confirm role** | Overall ≥ 80% AND every category ≥ 60% |
| ❌ **NOT PASS — not a fit** | Overall < 80% OR any category < 60% |

> **💡 Score ≠ final decision**
> The scoring framework gives a data-driven view. The final decision involves QA Lead + Engineering Lead + HR — numbers alone don't capture everything. Borderline cases (just above/below threshold) are discussed carefully before the decision is final.

---

## 7. What Happens at Each Outcome

At Day 60, the evaluation determines whether you're a good fit for Jitsu. There are 2 possible outcomes:

### ✅ PASS — Welcome to Jitsu officially!

Most QA hires pass probation. If you do:

- HR confirms your role officially.
- Celebrate with the team — you've earned it.
- You + QA Lead discuss the next 3-month growth focus: which skills to deepen, what new areas to explore.
- Regular sprint assignment going forward.

### ❌ NOT PASS

In some cases, the role isn't the right fit. If this happens:

- Your QA Lead explains the decision clearly with specific evidence from your work.
- Exit interview — you share your feedback so Jitsu improves the onboarding + hiring process.
- HR handles the official exit paperwork (notice period, last working day, etc.).
- Knowledge handoff for any in-progress tickets.

> **💡 If you feel uncertain about your trajectory**
>
> Don't wait until Day 60 to find out. Ask your QA Lead during Day 14 / Day 30 / Day 45 check-ins:
> - "Am I on track?"
> - "Which goal area should I focus on more?"
> - "Is there anything I'm missing that I should know about?"
>
> Early conversations almost always lead to better outcomes than late ones. The Day 30 review is a chance to course-correct — use it.

---

## 8. Tips to Succeed During Probation

Practical advice from QA Leads who've onboarded many QAs at Jitsu.

### 🎯 In Week 1–2

- Read the **Welcome Kit** + **2-Week Onboarding Plan** carefully — they're your roadmap.
- Set up all tooling on Day 1–2 (StrongDM, Postman, QMetry, Datadog access).
- Don't be afraid to ask 'dumb' questions — better to ask than guess.
- Post daily progress reports in the QA channel from Day 1 — it's a habit, not a punishment. Lead needs visibility.
- If you hit ANY blocker, ping Lead immediately — don't wait for daily standup.
- **Understand where Jitsu is heading**: Skim Section 9 in your first week. You don't need to study the career framework deeply — just know the direction. The skills you're building now feed directly into QI Engineering.

### 🎯 In Week 3–6

- Start picking up real tickets. It's OK if you're slow — focus on accuracy first.
- Use the 15-minute rule: if you're stuck > 15 min, ask your Buddy.
- After every ticket, review what went well + what you could improve.
- Apply feedback in the next ticket — show that you're learning.

### 🎯 In Week 7–8 (final stretch)

- Self-assess against the goals in Sections 2/3 — be honest with yourself.
- Bring questions to the Day 45 check-in: what should you focus on?
- Demonstrate growth — your Day 60 self should be visibly stronger than Day 1.
- Stay curious. Probation passed doesn't mean stop learning — it means you're trusted to keep going.

### 📚 Documentation discipline

- **When working on a feature** — first search Confluence for an existing feature doc. If found, update it as part of your ticket. If not, ask Lead if you should write one (it's almost always yes).
- **When you spot a new feature flag** — write a Confluence doc capturing: flag name, location (Consul / Firebase / `item_metadata` / etc.), default value, enable/disable conditions, scope (region/client/global). Check existing flag docs as templates.
- **Stale docs = quality bugs in disguise.** Outdated docs cause juniors to test wrong, devs to assume wrong behavior, and PMs to plan wrong features. Updating docs IS QA work.
- **Examples of good flag docs**: `claimsProcessingEnabled`, `SLA_VERSIONING_ENABLED`, `enable_require_id_scan`. Use these as templates when writing your own.

### 🤖 Using AI effectively

- **Use AI daily for**: drafting test cases from AC, summarizing long Confluence pages, explaining unfamiliar code areas, writing better bug repro steps, brainstorming edge cases.
- **Always verify** — AI can hallucinate Jitsu-specific terminology (fake driver types, wrong statuses, made-up API names). Cross-check against Confluence / code / asking the team.
- **Give AI context** — paste the ticket, the AC, relevant code, the Confluence page. AI without context = generic answers.
- **Iterate** — first prompt rarely perfect. Refine: "Make it more specific to Jitsu's `NEXT_DAYS` flow" / "Focus on edge cases only" / "Output as a markdown table".
- **Share your best prompts** with the team — what works for you probably works for others.

> **💡 The single best predictor of probation success**
>
> It's not technical brilliance. It's attitude + consistency:
> - Show up daily, do the work, report your progress
> - Ask questions when stuck
> - Apply feedback in the next ticket
> - Be honest about your gaps
> - Leverage AI as your assistant — not as authority
>
> QAs with great attitude almost always pass — even if their technical skills started slow.

---

## Section 9: Future of QA at Jitsu — Your Career Direction

> **TL;DR**: Jitsu is transitioning the QA function from one team into **two distinct disciplines** with separate career ladders. As a new Manual QA hire, your future is **QI (Quality Improvement) Engineering**. Your daily probation work in v1.1 is still relevant — it builds the foundation for QI. This section orients you to the direction so you can see where your career is heading.

### 9.1 Why this section exists

You may hear teammates or leadership mention "**QI Engineering**" or "**QS Engineering**", "**Deshi commands**", or the "**new QI System**". These are part of an organizational change announced in **March 2026** and rolling out over **9–18 months**. The probation goals you're working toward (Sections 2–8) are still 100% relevant — but it helps to know where the company is heading.

### 9.2 The transition in one sentence

> Jitsu is moving from a model where **QA is a phase** (devs build → QA tests) to a model where **quality is a property of the system** (continuously proven by automation, with QA enabling — not gatekeeping — quality).

### 9.3 Two new disciplines (effective Mar 2026)

| Discipline | Comes from | Focus |
|-----------|-----------|-------|
| **🔍 QI (Quality Improvement) Engineering** | **Manual QA team** (your path) | Exploratory testing, risk discovery, test design, acceptance criteria definition. Encoding domain knowledge through PRs with failing tests. |
| **🛠️ QS (Quality Systems) Engineering** | Automation QA team | Test frameworks, AI-assisted test generation, CI reliability, cross-system integration tests. Enabling devs to test effortlessly. |

**You (new Manual QA hire) are on the QI Engineering path.**

The 6-level QI career ladder is:

```
Entry  →  Developing  →  Senior  →  Staff  →  Sr Staff  →  Principal
  ↑          ↑
 You'll       After ~probation
  start
  here
```

### 9.4 What this means during your probation

**Right now (your daily work)**: Same as v1.1 of this document. Your team (Mobile or Webapp) is still using the **old QA model** — QMetry, manual verification, current Bug Reporting flow, etc. Probation goals in Sections 2–8 are your active rubric.

**On the horizon (months 6–18)**: The QI System (Deshi commands, `tc.md` test inventory, 7 Jira states) is currently piloted on the **Driver-Linehaul app only**. As rollout proceeds, your team will eventually adopt these practices. By the time you complete probation, you'll know the foundations — the new model builds on (not replaces) the QA skills you're learning.

### 9.5 7 Growth Dimensions of QI Engineering (preview)

When the new model fully rolls out, QI Engineers will be evaluated on these 7 dimensions. Many overlap with your current probation skills — they're **deepening**, not replacing, the foundation:

| # | QI Growth Dimension | Maps to v1.1 probation skills |
|---|---------------------|-------------------------------|
| 1 | **Exploratory Testing** | Quality of Work, Test Case Writing |
| 2 | **Risk Discovery** | Senior Cat 5 (Risk-based Test Approach), Quality of Work |
| 3 | **Test Design & Acceptance Criteria** | Test Case Writing, Feature Documentation Hygiene |
| 4 | **Developer Collaboration** | Soft Skills (cross-team), Proactive Issue Escalation |
| 5 | **Quality Advocacy** | Senior Cat 7 (Quality Advocacy), Risk Communication |
| 6 | **Risk Communication** | Soft Skills, Daily Progress Reports |
| 7 | **Jitsu Domain Knowledge** ⭐ | **First-class growth dimension** — your Cat 1 Domain Knowledge work is direct foundation |

> ⭐ **Domain Knowledge is a FIRST-CLASS dimension** in QI Engineering. The depth you're building now (Shipment Lifecycle, Assignment Lifecycle, Driver Types, cross-team interactions) is exactly what will set you apart as a QI Engineer.

### 9.6 The mindset shift to internalize early

Old model expected QA to be the **last line of defense** — catching bugs before release.

QI model expects you to **encode your knowledge into the codebase** — through clear acceptance criteria, test scenarios, and (eventually) PRs with failing tests that demonstrate quality gaps.

The shift in one sentence:

> **From observer to solver.**

Instead of "I found this bug, please fix it" → "I found this bug, here's a failing test that proves it, here's the analysis, here's the recommended fix."

You don't need to do this yet during probation. But knowing this is the direction helps you build the right habits early.

### 9.7 Reading list — go deeper

When you're ready (week 3–4 of probation, or after passing probation):

| Doc | Why |
|-----|-----|
| **[Quality Ownership and Collaboration](./career-framework/quality-ownership.md)** | The "why" — org-wide philosophy and transition context |
| **[QI Engineering — Career Ladder](./career-framework/qi-engineering.md)** | Your career ladder — 6 levels, 7 growth dimensions, stretch projects |
| **[QS Engineering — Career Ladder](./career-framework/qs-engineering.md)** | For context — what your Automation QA counterparts are evolving into |
| **[Jitsu QI System — Operational Reference](./jitsu-qi-system.md)** | What the new daily workflow looks like (currently Linehaul pilot only) |

### 9.8 Q&A

**Q: Does this mean my probation goals are about to change?**
A: No. Probation goals v1.2 are the same as v1.1 — no thresholds, categories, or skills changed. Section 9 is awareness only.

**Q: Will I have to use Deshi commands during probation?**
A: Only if you're assigned to the Linehaul team. Mobile (Driver/Outbound/Warehouse) and Webapp teams continue with the old model during your probation period. If you join Linehaul, your QA Lead will provide specific QI System onboarding.

**Q: What happens to my probation rubric when QI rolls out to my team?**
A: A new Probation Goals v2.0 aligned with QI Engineering growth dimensions will be issued for hires *after* the rollout. Your probation completes against v1.2 — what you signed up for at hire.

**Q: Is QS Engineering an option for me?**
A: Long-term yes — there's a documented lateral path from QI to QS (or to Software Development) for engineers who develop strong autonomous fixing + automation skills. But your immediate trajectory as Manual QA → QI Engineering.

**Q: Where do I ask questions about all this?**
A: Your QA Lead Hue Chu — directly on Slack DM. The career framework is new for everyone; questions are welcome.

---

> 💡 **Bottom line**: Section 9 is a **map of the road ahead** — not new homework. Keep executing against Sections 2–8. The skills you're building now compound directly into QI Engineering when the time comes.

---

**Good luck — you've got this! 🎯**

Welcome to the Jitsu QA Team.
