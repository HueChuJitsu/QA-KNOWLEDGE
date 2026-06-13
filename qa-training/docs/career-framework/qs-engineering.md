# Quality Systems Engineering

> **Version 1.0** • Last updated: May 27, 2026
> Source: [es.gojitsu.com/ecf/qs-engineering.html](https://es.gojitsu.com/ecf/qs-engineering.html)
> Effective: **March 2026**

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| **v1.0** | 2026-05-27 | Initial conversion of QS Engineering ECF doc into the qa-training repo. Source: `es.gojitsu.com/ecf/qs-engineering.html` (Mar 2026). |

---

## Reader's Guide

This document is the **Quality Systems Engineering discipline ladder** for Jitsu Engineering. It explains how the shared growth dimensions apply to engineers focused on test infrastructure, AI-assisted test generation, CI reliability, and enabling developers to test effortlessly.

Quality Systems Engineering is one of two quality-focused disciplines at Jitsu, alongside [Quality Improvement Engineering](./qi-engineering.md). While QI engineers proactively find and fix quality gaps, **QS engineers build the systems and tools that make quality achievable at scale**.

> 👥 **If You're Currently on the Automation QA Team**
>
> This is your discipline ladder. Quality Systems Engineering represents the **evolution of automation QA** at Jitsu. Your expertise in test frameworks, automation patterns, and cross-system testing is exactly what this role requires.
>
> The key shift is in **focus**: rather than writing every test yourself, you will increasingly **build systems that enable developers to write their own tests effortlessly** — with near-zero marginal effort to achieve full coverage.

---

## Role Scope and Philosophy

### 🛠️ Role Purpose

QS Engineers at Jitsu are responsible for **building the systems and tools that make quality achievable at scale**. They create test frameworks, AI-assisted test generation tools, and CI infrastructure that enable developers to write comprehensive tests with **near-zero marginal effort**.

QS also owns **cross-system integration tests** — tests that span multiple applications and require knowledge of how systems interact. This responsibility sits with QS because no single developer owns this cross-cutting knowledge; QS Engineers partner with QI Engineers for domain expertise about individual systems.

### 📈 Growth Expectations

Growth is expected at Jitsu, but it is **not linear**. Time at level varies based on role, opportunity, prior experience, and individual circumstances. What matters most is **sustained learning, increasing independence, and expanding impact over time**.

Consistent with Jitsu's engineering values, QS focuses on **leverage and enablement**, not test authorship volume. Effective QS work makes testing nearly effortless for everyone, turning quality from a bottleneck into a force multiplier.

---

## The Shift: From Volume to Leverage

Historically, automation QA at Jitsu has focused on writing and maintaining automated test suites — both cross-system integration tests and deep single-application tests. This has produced valuable test infrastructure and team members with strong technical skills in test automation.

However, the old model doesn't scale. Writing every test manually, even with automation skills, creates a bottleneck. The new model shifts focus **from volume to leverage**: QS Engineers build systems that make testing nearly effortless for everyone.

### Old Model vs New Model — QS

| ✗ Old Model (Automation QA) | ✓ New Model (QS Engineering) |
|-----------------------------|------------------------------|
| Write and maintain cross-system tests | Own cross-system integration tests (with QI domain input) |
| Write deep single-app test automation | Build frameworks and AI tools that enable dev testing |
| Be the experts who execute automation | Enable developers to write and own their tests |
| Own the tests end-to-end | Own the infrastructure, not every test |
| Tests may live in external validation repos | Tests reside within each repo itself |

> 💡 **The QS Mindset Shift**
>
> Instead of asking *"How do I test this system thoroughly?"* the QS question becomes:
>
> **"How do I make it trivially easy for any developer to test this system thoroughly?"**
>
> Success is measured by **developer testing velocity and org-wide coverage**, not by QS-authored test count.

> ⚠️ **This Is a Significant Transition**
>
> This shift will take time — Jitsu estimates **9–18 months** to fully transition. Progress is expected, but perfection is not. Jitsu commits to supporting this evolution through training, tooling investment, and reasonable timelines.
>
> See [Quality Ownership (Mar 2026)](./quality-ownership.md) for the full context on this transition.

---

## Key Responsibilities

### 1. AI-Assisted Test Generation

QS Engineers are responsible for building **next-generation generative AI tools** that enable engineering teams to write their own tests effortlessly. The target is **nearly zero marginal effort to achieve full test coverage**.

| Sub-area | Description |
|----------|-------------|
| 🤖 **AI Tooling** | Develop tools that instruct AI how to write tests in productive ways |
| 📦 **Repo Tuning** | Configure repos so tests are produced as a natural byproduct of development |
| 📋 **Best Practices** | Establish patterns that AI tools follow consistently across repos |
| 🔧 **Right Tech Stack** | Tools produce tests suited for the system under test, not the automator's preference |

### 2. Cross-System Integration Tests

Cross-system tests remain **primarily a QS responsibility** because they require knowledge of how multiple systems interact — knowledge that no single developer owns. QS Engineers:

- Own and maintain cross-system integration test suites
- Partner with QI Engineers for domain expertise about individual systems
- Ensure end-to-end flows (e.g., inbound → routing → delivery) have comprehensive coverage

### 3. Test Infrastructure & CI Reliability

- Test frameworks and architecture
- CI reliability and signal quality
- Flake reduction and test health
- Ensuring tests within repos follow consistent patterns

---

## Growth Dimensions

QS Engineering is evaluated across **7 growth dimensions**:

| # | Dimension | What it covers |
|---|-----------|----------------|
| 1 | **Test Architecture** | Design of test frameworks, structure, and patterns |
| 2 | **AI Tooling & Generation** | Building tools that instruct AI to write tests effectively |
| 3 | **Framework Development** | Building shared infrastructure (page objects, helpers, fixtures) |
| 4 | **Developer Enablement** | Making it trivially easy for devs to write comprehensive tests |
| 5 | **CI/CD Integration** | Reliable signal quality, flake reduction, build time |
| 6 | **Cross-System Testing** | E2E flows spanning multiple applications |
| 7 | **Quality Metrics** | Coverage, velocity, reliability tracking |

---

## Evaluation Focus

QS Engineers are evaluated on **leverage and enablement, not test authorship volume**.

### QS-owned metrics — things QS directly controls

| Metric | What it measures |
|--------|------------------|
| **Cross-system test coverage** | Coverage and reliability of integration tests that QS owns |
| **CI reliability** | Signal quality, flake rates, build times |
| **Framework adoption** | Are teams using the patterns and tools QS has built? |
| **Developer testing velocity** | How easily can developers write comprehensive tests using QS tooling? |

### Shared outcome — QS enables, but does not solely own

| Metric | Why shared |
|--------|------------|
| **Org-wide code and use-case coverage** | Coverage is a shared outcome across QS (enablement), QI (risk discovery and test design), and developers (test authorship). QS contributes by making coverage easy to achieve; rising coverage reflects effective QS enablement alongside developer and QI effort. |

---

## Level-by-Level Expectations

| Level | What Good Looks Like | Typical Scope |
|-------|---------------------|---------------|
| **Entry** | Learns test automation fundamentals and Jitsu's test infrastructure. Writes tests using established patterns. Partners with developers on test implementation. Learns AI-assisted test generation tools. | Individual test suites or components within a repo |
| **Developing** | Contributes to test frameworks and tooling. Configures AI-assisted test generation for repos. Helps developers write their own tests. Contributes to cross-system integration tests. | Test infrastructure for one or more teams' repos |
| **Senior** | Owns test frameworks for team's domain. Designs AI-assisted test generation for repos. Owns quality metrics (coverage, CI reliability). Owns cross-system integration tests for domain. | Test infrastructure and cross-system tests for a team or product area |
| **Staff** | Operates as team-level test platform owner. Shapes AI-assisted test generation tooling. Coaches developers and junior QS engineers. Evaluated by enablement impact: framework adoption, developer testing velocity, and CI reliability across team's repos. | Team-level test infrastructure and quality systems |
| **Sr Staff** | Operates as cross-team platform owner. Shapes test frameworks and AI tooling across teams. Establishes quality metrics and standards. Defines test infrastructure strategy. | Cross-team quality systems or shared test platforms |
| **Principal** | Sets company-wide direction for AI-powered quality systems. Ensures automation enables — not bottlenecks — velocity. Develops future quality systems leaders. Quality becomes a force multiplier. | Company-wide quality systems strategy and test infrastructure |

---

## Entry — QS Engineer (Entry)

Entry-level QS Engineers focus on **learning test automation fundamentals and Jitsu's test infrastructure**.

| Focus area | Description |
|------------|-------------|
| 📚 **Learning fundamentals** | Learns test automation concepts, Jitsu's test frameworks, and CI/CD systems. Understands how tests fit into the development workflow. |
| 🔧 **Contributing to tests** | Writes and maintains automated tests using established patterns. Contributes to existing test suites with guidance. |
| 🤝 **Beginning collaboration** | Partners with developers on test implementation. Learns how to support teams in writing their own tests. |
| 🤖 **Learning AI tools** | Uses AI-assisted test generation tools. Learns how to configure and tune them for different repos. |

**Scope**: Individual test suites or components within a repo

---

## Developing — QS Engineer

Developing QS Engineers operate **independently** on test automation work and begin building leverage.

| Focus area | Description |
|------------|-------------|
| 🏗️ **Framework contribution** | Contributes to test frameworks and tooling. Helps establish patterns that make testing easier for developers. |
| 🤖 **AI tool configuration** | Configures AI-assisted test generation for repos. Tunes tools to produce high-quality tests consistently. |
| 👥 **Developer enablement** | Helps developers write and maintain their own tests. Provides guidance on patterns and best practices. |
| 🔗 **Cross-system testing** | Contributes to cross-system integration tests. Partners with QI Engineers for domain knowledge. |

**Scope**: Test infrastructure for one or more teams' repos

---

## Senior — Senior QS Engineer

Senior QS Engineers **own test infrastructure for their domain** and drive developer testing adoption.

| Focus area | Description |
|------------|-------------|
| 🎯 **Framework ownership** | Owns test frameworks and patterns for team's domain. Ensures testing is nearly effortless for developers. |
| 🤖 **AI tooling leadership** | Designs and implements AI-assisted test generation for repos. Measures and improves coverage outcomes. |
| 📊 **Quality metrics** | Owns quality metrics for domain. Tracks coverage, CI reliability, and developer testing velocity. |
| 🔗 **Cross-system ownership** | Owns cross-system integration tests for domain. Partners with QI Engineers to encode domain knowledge. |

**Scope**: Test infrastructure and cross-system tests for a team or product area

---

## Staff — Staff QS Engineer

Staff QS Engineers **shape test infrastructure across their team** and coach others on quality systems.

| Focus area | Description |
|------------|-------------|
| 🏗️ **Test platform ownership** | Shapes AI-assisted test generation tooling used by team. Evaluated by code and use-case coverage across team's repos. |
| 👥 **Coaching** | Coaches developers and junior QS Engineers on test architecture and quality systems. Drives adoption of best practices. |
| 📈 **Scaling quality** | Ensures quality practices scale with team velocity. Identifies and removes testing bottlenecks. |
| 🤝 **QI collaboration** | Partners with QI Engineers to ensure domain knowledge is encoded into cross-system tests and acceptance criteria. |

**Scope**: Team-level test infrastructure and quality systems

---

## Sr Staff — Senior Staff QS Engineer

Senior Staff QS Engineers **shape quality systems across multiple teams** and drive org-wide test infrastructure.

| Focus area | Description |
|------------|-------------|
| 🌐 **Cross-team platform ownership** | Shapes test frameworks and AI tooling used across teams. Ensures near-zero marginal cost for comprehensive coverage. |
| 📊 **Org-wide quality metrics** | Establishes quality metrics and standards across teams. Partners with leadership on risk management. |
| 🔧 **Infrastructure strategy** | Defines test infrastructure strategy for the organization. Balances investment with delivery constraints. |

**Scope**: Cross-team quality systems or shared test platforms

---

## Principal — Principal QS Engineer

Principal QS Engineers **set company-wide direction** for quality systems and test infrastructure.

| Focus area | Description |
|------------|-------------|
| 🎯 **Quality systems vision** | Sets company-wide direction for AI-powered quality systems. Ensures automation enables — not bottlenecks — velocity. |
| 👥 **Leadership development** | Develops future quality systems leaders. Represents quality infrastructure perspectives in strategic decisions. |
| ⚡ **Velocity enablement** | Aligns quality systems strategy with business velocity goals. Quality becomes a force multiplier, not a constraint. |

**Scope**: Company-wide quality systems strategy and test infrastructure

---

## QS Engineering Principles

These **6 principles** guide QS Engineering work at Jitsu:

| # | Principle | What it means |
|---|-----------|---------------|
| **1** | **Tests in Repos** | All tests for a repo reside within the repo itself. **No external validation repos.** |
| **2** | **Tuned Repos** | Repos are configured so tests are produced as a **natural byproduct of development**. |
| **3** | **Right Tech Stack** | Tools produce tests suited for the **system under test**, not the automator's preference. |
| **4** | **Best Practices** | AI tools follow patterns established by QS Engineers within each repo. |
| **5** | **Dev Accountability** | Engineering teams are accountable for using QS tools to generate tests as part of their work. |
| **6** | **Coverage Metrics** | QS Engineers are evaluated by **code and use-case coverage** across all supported repos. |

---

## Stretch Projects

Stretch projects help engineers grow from one level to the next. Below are illustrative projects organized by transition.

### Entry → Developing: Building Foundation

#### Configure AI Test Generation for a Repo
Set up and tune AI-assisted test generation for a repo. Establish patterns that produce high-quality tests consistently.

- **Support**: Senior QS + Developers
- **Outcome**: Developers can generate tests with near-zero effort

#### Improve Test Framework Patterns
Identify friction in test authoring and improve patterns to make it easier. Reduce boilerplate, add helpers, improve documentation.

- **Support**: Senior QS
- **Outcome**: Measurable reduction in test authoring effort

### Developing → Senior: Ownership & Metrics

#### Own Cross-System Integration Tests
Take ownership of cross-system integration tests for a domain. Partner with QI Engineers for domain knowledge. Ensure comprehensive coverage of end-to-end flows.

- **Support**: QI Engineer + Developers
- **Outcome**: Reliable cross-system test coverage

#### Drive Coverage Improvement
Own coverage metrics for a team's repos. Identify gaps, improve tooling, and measure outcomes. Target significant improvement in code and use-case coverage.

- **Support**: EM + Developers
- **Outcome**: Measurable coverage improvement

### Senior → Staff: Platform Leadership

#### Build AI Test Generation Platform
Build or significantly extend AI-assisted test generation for team's repos. Target: developers can achieve comprehensive coverage with near-zero marginal effort.

- **Support**: DevOps + Developers
- **Outcome**: Significant increase in coverage with less effort

#### CI Reliability Initiative
Lead initiative to improve CI reliability for team. Reduce flakes, improve signal quality, decrease build times.

- **Support**: DevOps + Engineering leadership
- **Outcome**: Measurable improvement in CI reliability

### Staff → Sr Staff: Cross-Team Impact

#### Scale AI Test Generation Across Teams
Scale AI-assisted test generation across multiple repos and teams. Measured by code and use-case coverage improvements across the org.

- **Support**: DevOps + Engineering leadership
- **Outcome**: Org-wide coverage improvement

#### Define Quality Systems Standards
Define and drive adoption of test infrastructure standards across teams. Ensure consistency in patterns, tooling, and metrics.

- **Support**: Engineering leadership
- **Outcome**: Consistent quality systems across org

---

## Notes on Evaluation and Growth

Evaluation focuses on **leverage and enabling outcomes**, not personal test authorship volume. Success is measured by org-wide coverage improvements, developer testing velocity, and CI reliability — **not by how many tests QS engineers write themselves**.

Advancing in level requires not just building more infrastructure, but avoiding behaviors that create hidden costs.

### ⚠️ Anti-patterns that block growth

- ❌ **Building frameworks that only you can maintain**
- ❌ Writing tests that should be owned by developers
- ❌ Optimizing for QS productivity rather than developer enablement
- ❌ **Hoarding knowledge** about test infrastructure rather than documenting and sharing it
- ❌ Acting senior by tenure rather than judgment

**Technical automation skills are a first-class growth dimension** for QS. Deep expertise in test frameworks, CI/CD, and AI-assisted tooling meaningfully increases impact. At the same time, long-term growth requires pairing that expertise with **developer enablement** — building systems that others can use effectively.

---

## Jitsu's Commitment

This transition is significant. Jitsu commits to supporting it through:

| Support | Description |
|---------|-------------|
| 📚 **Training** | AI tooling, framework development, and platform thinking |
| 💰 **Investment** | Resources for building AI-assisted test generation infrastructure |
| 👥 **Collaboration** | Partnership with QI Engineers for domain knowledge encoding |
| ⏰ **Time** | 9–18 months to transition; progress expected, not perfection |

---

## 🤝 Our Promise

This shift is **not** about reducing QS headcount or devaluing automation expertise. It is about **amplifying your impact**:

- Your work enables every developer to achieve comprehensive coverage
- Your leverage multiplies across the entire organization
- Your success is measured by org-wide quality outcomes
- **Quality becomes a capability of the organization, not a bottleneck**

---

## Related Documents

- [Quality Ownership and Collaboration](./quality-ownership.md) — Org-wide philosophy and transition context
- [QI Engineering — Career Ladder](./qi-engineering.md) — Sibling discipline for Manual QA team
- [Probation Goals — Junior + Senior QA](../jitsu-qa-probation-goals.md) — Current probation framework

---

*This document is a mirror of the official Jitsu ECF page. For the authoritative version, see [es.gojitsu.com/ecf/qs-engineering.html](https://es.gojitsu.com/ecf/qs-engineering.html).*
