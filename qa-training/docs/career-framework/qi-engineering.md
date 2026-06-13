# Quality Improvement Engineering

> **Version 1.0** • Last updated: May 27, 2026
> Source: [es.gojitsu.com/ecf/qi-engineering.html](https://es.gojitsu.com/ecf/qi-engineering.html)
> Effective: **March 2026**

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| **v1.0** | 2026-05-27 | Initial conversion of QI Engineering ECF doc into the qa-training repo. Source: `es.gojitsu.com/ecf/qi-engineering.html` (Mar 2026). |

---

## Reader's Guide

This document is the **Quality Improvement Engineering discipline ladder** for Jitsu Engineering. It explains how the shared growth dimensions apply to engineers focused on exploratory testing, risk discovery, test design, and proactive quality improvement.

Quality Improvement Engineering is one of two quality-focused disciplines at Jitsu, alongside [Quality Systems Engineering](./qs-engineering.md). While QS engineers build the systems and tools that enable testing at scale, **QI engineers are the "quality special forces"** — proactively seeking out and closing quality gaps across our systems.

> 👥 **If You're Currently on the Manual QA Team**
>
> This is your discipline ladder. Quality Improvement Engineering represents the **evolution of manual QA** at Jitsu. Your deep domain knowledge, exploratory testing skills, and understanding of customer workflows are exactly what this role requires.
>
> The key shift is in **how you express that expertise**: rather than only observing and reporting issues, you will increasingly **identify and solve them autonomously** — submitting pull requests with failing tests that prove the issue, not just Jira tickets.

---

## Role Scope and Philosophy

### 🔍 Role Purpose

QI Engineers at Jitsu are responsible for **proactively finding and closing quality gaps** across our systems. They combine deep domain knowledge with exploratory testing skills to discover risks that automated tests miss, then encode that knowledge into testable form — **through PRs with failing tests, not just tickets**.

### 📈 Growth Expectations

Growth is expected at Jitsu, but it is **not linear**. Time at level varies based on role, opportunity, prior experience, and individual circumstances. What matters most is **sustained learning, increasing independence, and expanding impact over time**.

Consistent with Jitsu's engineering values, QI focuses on **knowledge acquisition and risk reduction**, not merely defect detection. Effective QI work turns unknowns into knowns and encodes that understanding into tests, acceptance criteria, and process improvements.

---

## The Shift: From Observer to Solver

Historically, manual QA at Jitsu has focused on requirements study, test planning, deep system understanding, and careful execution. This has produced QA team members with **exceptional Jitsu domain knowledge** — often rivaling or exceeding that of software engineers. This expertise is extremely valuable and remains critical to our success.

However, the old model limited how that expertise could be applied. As manual QA workers, your knowledge lived in your head and was expressed through execution. As Quality Improvement Engineers, you **encode that knowledge into acceptance criteria, test scenarios, and risk assessments** that shape what developers build and test.

### Old Model vs New Model — QI

| ✗ Old Model (Manual QA) | ✓ New Model (QI Engineering) |
|-------------------------|------------------------------|
| Observe and report issues | Encode domain knowledge into acceptance criteria and test scenarios |
| Create Jira tickets with reproduction steps | Partner with developers to ensure comprehensive test coverage |
| Wait for engineering to prioritize and fix | Exploratory testing focused on unknown risks |
| Manual regression testing as primary activity | Communicate quality risks and influence priorities |
| Knowledge expressed through execution | Use AI tools to prove issues with failing tests (stretch goal) |

> ⚠️ **This Is a Significant Transition**
>
> This shift will take time — Jitsu estimates **9–18 months** to fully transition. Progress is expected, but perfection is not. Jitsu commits to supporting this evolution through training, AI tooling, mentorship, and reasonable timelines.
>
> See [Quality Ownership (Mar 2026)](./quality-ownership.md) for the full context on this transition.

---

## The Key Asset: Domain Knowledge

QI Engineers at Jitsu have deep knowledge of our systems, workflows, and edge cases. This knowledge is extremely valuable and is a **first-class growth dimension** in this discipline.

> 💡 **How Domain Knowledge Is Expressed**
>
> **Before**: Knowledge expressed through manual test execution — you knew the system, so you tested it.
>
> **After**: Knowledge encoded into acceptance criteria, test scenarios, and automated coverage — your expertise shapes what gets tested and how, across the entire organization.

This shift doesn't diminish the value of domain knowledge. It **amplifies** it. Instead of one person's knowledge being limited to what they can manually execute, it becomes encoded and scalable.

### QI Engineers Collaborate With

| Partner | Collaboration pattern |
|---------|----------------------|
| **Developers** | Providing test scenarios, acceptance criteria, and risk discovery for their features |
| **QS Engineers** | Encoding domain knowledge into cross-system integration tests |

---

## Key Responsibilities

### 1. Exploratory Testing & Risk Discovery

QI Engineers proactively find quality gaps that automated tests miss. They use deep domain knowledge to identify where systems are likely to fail under real-world conditions.

| Sub-area | Description |
|----------|-------------|
| 🔍 **Risk-Based Testing** | Focus testing effort on areas of highest risk and uncertainty |
| 🎯 **Edge Case Discovery** | Find the scenarios that developers and automated tests miss |
| ⚠️ **Failure Mode Analysis** | Understand how systems fail and what conditions trigger failures |

### 2. Test Design & Acceptance Criteria

QI Engineers translate domain knowledge into clear acceptance criteria and test scenarios that shape what developers build and test.

- Define acceptance criteria that capture real-world requirements and edge cases
- Design test scenarios that developers can implement as automated tests
- Partner with developers to ensure comprehensive coverage **before features ship**

### 3. Domain Knowledge Encoding

QI Engineers capture institutional knowledge about Jitsu's systems, workflows, and failure modes in **durable, reusable form**.

- Document edge cases, known risks, and failure patterns for product areas
- Create reusable test scenario templates that encode domain expertise
- Partner with QS Engineers to encode domain knowledge into cross-system integration tests

---

## Growth Dimensions

QI Engineering is evaluated across **7 growth dimensions**:

| # | Dimension | What it covers |
|---|-----------|----------------|
| 1 | **Exploratory Testing** | Finding unknown risks through hands-on system exploration |
| 2 | **Risk Discovery** | Identifying where systems are likely to fail |
| 3 | **Test Design & Acceptance Criteria** | Translating risk into clear AC + test scenarios |
| 4 | **Developer Collaboration** | Partnering effectively with engineering on test authoring |
| 5 | **Quality Advocacy** | Influencing what gets prioritized based on quality risk |
| 6 | **Risk Communication** | Conveying quality risks clearly to stakeholders |
| 7 | **Jitsu Domain Knowledge** ⭐ | First-class dimension — deep understanding of our systems and workflows |

---

## Evaluation Focus

QI Engineers are evaluated on **quality impact, not activity**. Key metrics include:

| Metric | What it measures |
|--------|------------------|
| **Domain knowledge encoding** | Is your expertise being captured in acceptance criteria, test scenarios, and documentation? |
| **Test design quality** | Are acceptance criteria clear? Do test scenarios catch real risks? |
| **Developer collaboration** | Are developers seeking you out for test scenarios and risk assessment? |
| **Risk discovery** | Are you finding quality gaps that others miss through exploratory testing? |
| **Quality advocacy** | Are you influencing what gets prioritized based on quality risk? |

---

## Level-by-Level Expectations

| Level | What Good Looks Like | Typical Scope |
|-------|---------------------|---------------|
| **Entry** | Learns Jitsu's systems, workflows, and quality practices. Executes exploratory testing with guidance. Reports defects clearly. Partners with developers on test scenarios. Learns AI tools for test generation. | Individual features or small workflows |
| **Developing** | Designs test scenarios and acceptance criteria independently. Conducts exploratory testing to find issues developers missed. Partners effectively with developers on test authoring. Begins encoding domain knowledge into reusable test patterns. | Features, services, or workflows within a team |
| **Senior** | Defines test strategies for complex systems. Communicates quality risks clearly and influences priorities. Encodes deep domain knowledge into acceptance criteria that shape what gets built. Mentors others in test design and risk thinking. | Major features, cross-cutting workflows, or critical releases |
| **Staff** | Operates as team-level quality leader. Establishes testing standards and acceptance criteria patterns. Coaches developers and junior QI engineers. Partners with QS on cross-system test coverage. | Team-level quality initiatives and domain expertise |
| **Sr Staff** | Operates as cross-team quality leader. Establishes standards across teams. Leads org-wide risk reduction initiatives. Influences product and engineering decisions to reduce systemic risk. | Cross-team quality initiatives or multiple product areas |
| **Principal** | Sets company-wide quality and risk strategy. Influences engineering and release practices. Develops future quality leaders. Leverages deep domain expertise to protect customer trust at scale. | Company-wide quality strategy and foundational practices |

---

## Entry — QI Engineer (Entry)

Entry-level QI Engineers focus on **learning Jitsu's systems, workflows, and quality practices**.

| Focus area | Description |
|------------|-------------|
| 📚 **Learning the domain** | Learns Jitsu's core business flows, systems, and data. Understands how defects affect customers and operations. |
| 🔍 **Exploratory testing with guidance** | Executes exploratory testing on assigned features. Reports defects clearly and reproducibly. |
| 🤝 **Beginning collaboration** | Partners with developers on test scenarios. Learns how acceptance criteria translate to test cases. |
| 🛠️ **Learning AI tools** | Learns to use AI tools for test generation and issue investigation. Can read and understand existing automated tests. |

**Scope**: Individual features or small workflows

---

## Developing — QI Engineer

Developing QI Engineers operate **independently** on routine quality work and encode their domain knowledge into test artifacts.

| Focus area | Description |
|------------|-------------|
| 📋 **Test design** | Designs test scenarios and acceptance criteria for features. Identifies edge cases and risk areas proactively. |
| 🔍 **Risk discovery** | Conducts exploratory testing to find issues developers missed. Focuses on unknown risks, not obvious regressions. |
| 📚 **Domain knowledge encoding** | Translates system knowledge into reusable test patterns and acceptance criteria. Documents edge cases for future reference. |
| 👥 **Developer collaboration** | Partners effectively with developers on test authoring. Provides scenarios and risk insights; developers implement. |

**Scope**: Features, services, or workflows within a team's ownership

---

## Senior — Senior QI Engineer

Senior QI Engineers are **trusted to assess and manage quality risk independently**. They shape what gets tested through deep domain expertise.

| Focus area | Description |
|------------|-------------|
| 🎯 **Test strategy** | Defines test strategies for complex systems. Balances exploratory testing with developer-written automation. |
| 📚 **Domain expertise** | Demonstrates strong Jitsu domain knowledge. Encodes that knowledge into acceptance criteria that shape what developers build and test. |
| 💬 **Quality advocacy** | Communicates quality risks clearly to stakeholders. Influences what gets prioritized based on risk assessment. |
| 👥 **Mentoring** | Mentors others in test design, risk thinking, and domain knowledge. Helps junior QI engineers develop expertise. |

**Scope**: Major features, cross-cutting workflows, or critical releases

---

## Staff — Staff QI Engineer

Staff QI Engineers **influence quality practices across their team** and serve as domain SMEs.

| Focus area | Description |
|------------|-------------|
| 🏗️ **Team-level quality leadership** | Establishes testing standards and acceptance criteria patterns for team's domain. Ensures quality practices scale with velocity. |
| 👥 **Coaching** | Coaches developers and junior QI Engineers on test design, risk thinking, and quality practices. |
| 🎯 **Domain SME** | Acts as domain expert for quality. Influences product design to reduce risk at the source. |
| 🤝 **QS collaboration** | Partners with QS Engineers to encode domain knowledge into cross-system integration tests. |

**Scope**: Team-level quality initiatives and domain expertise

---

## Sr Staff — Senior Staff QI Engineer

Senior Staff QI Engineers **shape quality practices across multiple teams** and lead org-wide risk reduction.

| Focus area | Description |
|------------|-------------|
| 🌐 **Cross-team quality leadership** | Establishes standards for test design and acceptance criteria across teams. Partners with leadership on risk management. |
| 🔍 **Org-wide risk reduction** | Leads quality initiatives for complex, cross-team systems. Identifies and addresses systemic quality gaps. |
| 💡 **Product influence** | Influences product and engineering decisions to reduce systemic risk. Quality considerations shape design. |

**Scope**: Cross-team quality initiatives or multiple product areas

---

## Principal — Principal QI Engineer

Principal QI Engineers **shape how Jitsu approaches quality at the company level**.

| Focus area | Description |
|------------|-------------|
| 🎯 **Quality strategy** | Defines long-term quality and risk strategy. Influences company-wide engineering and release practices. |
| 👥 **Leadership development** | Develops future quality leaders. Represents quality perspectives in strategic decisions. |
| 🛡️ **Customer trust** | Leverages deep domain expertise to protect customer trust at scale. Quality becomes a competitive advantage. |

**Scope**: Company-wide quality strategy and foundational practices

---

## When Issues Are Too Big

Not every issue can be solved autonomously by a QI Engineer. When the discovered problem is too large or complex, QI Engineers escalate to the engineering team with:

| Element | Description |
|---------|-------------|
| 🧪 **Failing Tests** | A PR that includes failing tests demonstrating the issue |
| 📋 **Analysis** | An analysis of the problem and recommended solution |
| ⚠️ **Assessment** | An assessment of the severity and urgency of the issue |

This is **still a significant improvement** over the old model — the engineering team receives a PR with tests that prove the issue, not just a Jira ticket with reproduction steps.

---

## Stretch Projects

Stretch projects help engineers grow from one level to the next. Below are illustrative projects organized by transition.

### Entry → Developing: Building Foundation

#### Own Test Design for a Feature
Define acceptance criteria and test scenarios for a feature (e.g., driver booking flow or recipient notifications). Partner with developer to ensure tests get written.

- **Support**: Developer + Senior QI
- **Outcome**: Feature ships with comprehensive automated coverage

#### Find and Document Issues Others Missed
Conduct exploratory testing on a recently shipped feature. Document edge cases and risks that automated tests don't cover. Create failing tests that prove the issues.

- **Support**: Senior QI
- **Outcome**: Quality gaps identified and proved with tests

### Developing → Senior: Strategic Test Design

#### Define Cross-System Test Strategy
Define test strategy for a cross-system flow (e.g., booking → assignment → warehouse pickup → delivery). Coordinate with multiple dev teams on acceptance criteria.

- **Support**: Developers + QS Engineer
- **Outcome**: Coherent acceptance criteria across system boundaries

#### Build Domain Knowledge Repository
Document edge cases, failure modes, and risk patterns for a product area. Create reusable test scenario templates that capture your domain expertise.

- **Support**: Senior QI + Tech Lead
- **Outcome**: Domain knowledge encoded and reusable

#### Fix Quality Issues End-to-End (Stretch)
Find a quality issue, write failing tests that prove it, and submit a PR with the fix — not just a Jira ticket. Use AI tools to assist with the implementation.

- **Support**: Senior QI + Developer
- **Outcome**: Quality issue solved autonomously

### Senior → Staff: Domain Leadership

#### Own Team Quality Standards
Define and drive adoption of test design standards within your team's domain. Ensure developers understand and follow quality practices.

- **Support**: EM + Developers
- **Outcome**: Consistent quality standards across team

#### Encode Domain Knowledge for Cross-System Tests
Partner with QS Engineers to encode your domain expertise into cross-system integration tests. Your knowledge shapes what gets tested across system boundaries.

- **Support**: QS Engineer
- **Outcome**: Critical cross-system scenarios have comprehensive coverage

### Staff → Sr Staff: Organizational Influence

#### Create Cross-Team Quality Standards
Create and drive adoption of test design and acceptance criteria standards across multiple teams.

- **Support**: Engineering leadership
- **Outcome**: Consistent quality standards across org

#### Influence Product Design for Quality
Embed quality considerations into product design. Identify design patterns that reduce defects at the source.

- **Support**: Product + Tech Leads
- **Outcome**: Earlier defect prevention

---

## Notes on Evaluation and Growth

Evaluation focuses on **sustained quality impact**, clarity of communication, and the ability to turn quality insights into durable improvements. Finding fewer bugs because systems are better designed and better tested is a **success, not a failure**.

Advancing in level requires not just finding more issues, but increasingly **encoding your expertise and shaping what gets tested**.

### ⚠️ Anti-patterns that block growth

- ❌ Closing tickets without understanding root cause
- ❌ Reporting issues repeatedly without encoding knowledge into tests
- ❌ **Hoarding domain knowledge** rather than encoding it into acceptance criteria and test scenarios
- ❌ Prioritizing issue volume over issue impact and prevention
- ❌ Acting senior by tenure rather than judgment

Jitsu domain knowledge is a **first-class growth dimension** for QI. Deep understanding of our systems, workflows, and operational realities meaningfully increases impact. At the same time, long-term growth requires pairing that expertise with the ability to **encode it into scalable, testable form**.

---

## Jitsu's Commitment

This transition is significant. Jitsu commits to supporting it through:

| Support | Description |
|---------|-------------|
| 📚 **Training** | Test design, AI tools, and autonomous problem-solving patterns |
| 🤖 **AI Tooling** | Tools that enable you to identify and solve issues autonomously |
| 👥 **Mentorship** | Pairing and coaching for developing new skills |
| ⏰ **Time** | 9–18 months to transition; progress expected, not perfection |

---

## 🤝 Our Promise

This shift is **not** about reducing QA or devaluing what manual QA has contributed. It is about **amplifying your impact**:

- Your domain knowledge becomes encoded and scalable
- Your expertise shapes what gets tested, not just how
- You solve problems, not just report them
- Your work enables the entire org to ship with confidence

---

## Growth Beyond QI

As Jitsu's AI-assisted development tools mature, QI Engineers who are interested will have increasing opportunities to **solve issues autonomously** — finding bugs, writing failing tests, and submitting fixes.

This capability is **not required** to succeed or advance in QI. The core QI role is about domain knowledge, risk discovery, and test design. However, for engineers who develop strong autonomous fixing skills, this opens pathways to:

| Path | Description |
|------|-------------|
| **Lateral move to Software Development** | Leveraging domain expertise plus coding fluency |
| **Lateral move to Quality Systems Engineering** | Building test infrastructure and frameworks |

Jitsu will support engineers who want to pursue these paths through training, mentorship, and stretch project opportunities. **This is an option, not an expectation.**

---

## Related Documents

- [Quality Ownership and Collaboration](./quality-ownership.md) — Org-wide philosophy and transition context
- [QS Engineering — Career Ladder](./qs-engineering.md) — Sibling discipline for Automation team
- [Probation Goals — Junior + Senior QA](../jitsu-qa-probation-goals.md) — Current probation framework

---

*This document is a mirror of the official Jitsu ECF page. For the authoritative version, see [es.gojitsu.com/ecf/qi-engineering.html](https://es.gojitsu.com/ecf/qi-engineering.html).*
