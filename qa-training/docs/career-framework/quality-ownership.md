# Quality Ownership and Collaboration

> **Version 1.1** • Last updated: May 27, 2026
> Source: [es.gojitsu.com/ecf/quality-ownership.html](https://es.gojitsu.com/ecf/quality-ownership.html)
> Effective: **March 2026**

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| **v1.0** | 2026-05-27 | Initial conversion of Quality Ownership ECF doc into the qa-training repo. Source: `es.gojitsu.com/ecf/quality-ownership.html` (Mar 2026). |
| **v1.1** | 2026-05-27 | Jitsu-specific refinement of Developer Responsibilities: AC is **read carefully** (not defined) by devs — AC ownership lives with PO + QI through Three Amigos. Added "Verify code against test cases before QS review" responsibility. Updated comparison table + transition guidance accordingly. |

---

## ⚠️ This Is a Significant Shift

This document describes a **substantial change** in how developers and QA work together at Jitsu. It affects expectations, responsibilities, and collaboration patterns for both groups. Jitsu is committed to this transition, and recognizes it will take **9–18 months** to fully realize.

What matters most during this period is **momentum and commitment** — not immediate perfection.

---

## A New Model for Quality at Jitsu

### Why We're Changing

The way Jitsu has historically approached quality was designed for a slower, more manual world. It served us well for a time, but it no longer scales with our velocity, our use of AI-assisted development tools, or the complexity of our systems.

### The Problems We're Solving

| Problem | Description |
|---------|-------------|
| 🚧 **Manual QA as a bottleneck** | Regression testing doesn't scale. Developers avoid large or risky changes because QA can't validate them within a sprint. |
| 🤾 **Testing as a separate phase** | When testing happens after development rather than alongside it, quality becomes someone else's problem. The structure itself discourages shared ownership. |
| 📅 **Sprint-oriented quality thinking** | Tickets are considered "done" when coded, not when proven safe. Releases are gated by QA availability rather than system confidence. |
| 🤖 **AI increases risk without matching quality evolution** | AI-assisted coding dramatically increases change velocity. Without high automation coverage, AI-generated code erodes trust. |

### The Deeper Issues

Beneath these problems are structural and cultural patterns that need to change:

- **Quality is treated as a phase.** Devs build, QA tests, bugs are found late. This makes quality a stage rather than a property of the system.
- **QA is positioned as a gate.** QA is implicitly responsible for catching regressions, preventing bad releases, and absorbing test debt. Over time, this structure can limit developer ownership and stretch QA capacity.
- **Manual execution is confused with quality value.** Manual QA knowledge is deep and valuable, but it is expressed through execution rather than encoded intent. That knowledge doesn't compound or scale.
- **Ownership is misaligned.** Devs optimize for code completion. QA optimizes for defect detection. No one optimizes for continuous proof of correctness.

---

## The New Mental Model

> 💡 **The Core Insight**
>
> In an AI-augmented organization, **code is cheap, rewrites are cheap, but regressions are expensive.** Therefore:
>
> - **Automation is the primary trust mechanism**
> - **Tests are assets, not overhead**
> - **Quality must be continuously provable**
>
> **Quality is not something we check. It is something the system demonstrates.**

This shift has implications for everyone in engineering.

---

## What Changes for Developers

In the old model, developers wrote unit tests (sometimes) and handed work to QA for validation. In the new model, **developers own proving that their code works and keeps working.**

### Old Model vs New Model — Developers

| ✗ Old Model | ✓ New Model |
|-------------|-------------|
| Write code and write unit tests | Read acceptance criteria carefully before coding |
| Hand off to QA for validation | Write and maintain automated tests for changes |
| Acceptance criteria defined after implementation | Collaborate with QA on test design and edge cases |
| "Done" when happy path is checked | "Done" when the system can continuously prove it works |
| Testing is largely owned by QA | Proving correctness is part of the job |

### Developer Responsibilities

Developers **are** responsible for:

- **Reading acceptance criteria carefully before coding** — AC is finalized by PO + QI + Tech Lead during Three Amigos; developers consume it, not define it
- Writing and maintaining automated tests for their changes — based on the test cases and AC reviewed during Three Amigos
- **Verifying code against the test cases** before pushing the PR for QS team review — quality is proven first, not delegated downstream
- Ensuring coverage for normal scenarios and critical edge cases
- Using AI tools to generate and maintain tests efficiently
- Collaborating with QA to identify risk areas and edge cases

Developers are **not** responsible for:

- Defining acceptance criteria from scratch (AC ownership lives with PO + QI)
- Exhaustive exploratory testing
- Test framework architecture
- Organization-wide quality strategy

> 📌 **Jitsu-specific clarification**
> The ECF generally states developers "define AC before coding". At Jitsu, AC is **finalized through the Three Amigos process** (PO + QI + Tech Lead). Developers receive reviewed, agreed-upon AC and are accountable for honoring it in code + tests before handing off to QS review.

> 🔧 **AI Makes This Possible**
>
> With modern AI tools, the marginal cost of writing comprehensive tests approaches zero. This changes the tradeoff calculus: teams can now achieve coverage that was previously impractical. When tests are missing, it should be because we **intentionally scoped them out** — not because time ran short.

---

## What Changes for QA

This shift is substantial for QA team members. The old model placed QA as a regression safety net and sprint gate. The new model repositions QA as **quality enablers** whose deep system knowledge becomes a strategic asset.

### Old Model vs New Model — QA

| ✗ Old Model | ✓ New Model |
|-------------|-------------|
| Manual regression testing | Exploratory testing and risk discovery |
| Sprint gate / release blocker | Test design and acceptance criteria definition |
| Last line of defense | Collaboration with devs on test authoring |
| Write automation tests separately from devs | Proactively finding and fixing quality gaps |
| Knowledge expressed through execution | Knowledge encoded into tests and tooling |

---

## Two Quality Disciplines

To support this transition, Jitsu is establishing **two distinct quality-focused disciplines** with their own career ladders:

### 🔍 Quality Improvement (QI)

Exploratory testing, risk discovery, test design, acceptance criteria definition. Proactively finding and fixing quality gaps. Encodes domain knowledge into testable form via **PRs with failing tests** — not just tickets.

### 🛠️ Quality Systems (QS)

Test frameworks, AI-assisted test generation, CI reliability, cross-system integration tests. Builds systems that make testing effortless for developers. Evaluated by **enablement impact** (framework adoption, developer testing velocity, CI reliability), **not test authorship volume**.

> 📌 These are separate disciplines with distinct career ladders, reflecting the different skill sets, responsibilities, and growth paths.
>
> - If you're currently on the **Manual QA team** → see [Quality Improvement Engineering](https://es.gojitsu.com/ecf/qi-engineering.html)
> - If you're on the **Automation QA team** → see [Quality Systems Engineering](https://es.gojitsu.com/ecf/qs-engineering.html)

---

## For Today's Automation QA Team → Quality Systems

Jitsu's current Automation QA team has built deep expertise in two distinct areas:

- **Cross-system integration tests** — Tests that span multiple applications and require knowledge of how systems interact (e.g., end-to-end flows from inbound through routing to delivery)
- **Deep single-system automated tests** — Comprehensive automated test suites for individual applications, covering complex business logic and edge cases

In the new model, this team transitions to **Quality Systems (QS)**. Here's what changes:

### Current State vs Future State — QS

| Current State | Future State (QS) |
|---------------|-------------------|
| Write and maintain cross-system tests | Continue supporting cross-system tests (with QI collaboration for domain expertise) |
| Write deep single-app test automation | Create leverage: frameworks, patterns, and AI tools that make testing effortless for devs |
| Be the experts who execute automation | Enable developers to write and own their tests |
| Own the tests end-to-end | Own the **infrastructure**, not every test |

**Cross-system tests remain important** because they require multiple domain knowledge areas that no single developer owns. QS will continue to support these, working with QI engineers who bring exploratory insight and risk discovery across system boundaries.

**Single-system test ownership shifts to developers.** Rather than QS writing every deep test, QS creates the leverage — frameworks, AI-assisted generation tools, patterns, and documentation — that makes it nearly effortless for developers to write comprehensive tests for their own systems.

> 🔧 **The QS Mindset Shift**
>
> Instead of asking *"How do I test this system thoroughly?"* the QS question becomes:
>
> **"How do I make it trivially easy for any developer to test this system thoroughly?"**
>
> Success is measured by **enablement impact** — framework adoption, developer testing velocity, and CI reliability — not by QS-authored test count.

---

## The Key Asset: Domain Knowledge

QA team members at Jitsu have deep knowledge of our systems, workflows, and edge cases — often exceeding that of developers. This knowledge is extremely valuable. The shift is in **how we express and leverage it**:

- **Before**: Knowledge expressed through manual test execution
- **After**: Knowledge encoded into acceptance criteria, test scenarios, and automated coverage

QA's role becomes **defining what should be tested and why**, rather than being the ones who execute all the tests.

---

## How Developers and Quality Engineers Work Together

The new model requires **genuine collaboration**, not handoffs. Each discipline contributes differently:

### Developers + QI Engineers

| Pattern | Description |
|---------|-------------|
| 📋 **QI provides scenarios, devs implement tests** | QI Engineers identify edge cases, risk areas, and acceptance criteria. Developers (with AI assistance) implement the automated tests. |
| 🔍 **Exploratory testing finds surprising issues** | QI focuses on unknown risks and real-world usage patterns — not regressions that automation should catch. |
| 🐛 **PRs not tickets** | QI Engineers submit PRs with failing tests to demonstrate quality gaps — not just tickets describing issues. This encodes domain knowledge directly into the codebase. |
| 🔄 **Acceptance criteria before coding** | QI helps define what "done" looks like before implementation begins. Tests are not an afterthought. |

### Developers + QS Engineers

| Pattern | Description |
|---------|-------------|
| 🤖 **QS provides tooling, devs write tests** | QS Engineers build AI-assisted test generation tools and frameworks. Developers use these to achieve comprehensive coverage with minimal effort. |
| 📦 **Tuned repos** | QS configures repos so tests are produced as a natural byproduct of development. Best practices are encoded into the tooling. |
| 🔗 **Cross-system tests** | QS owns cross-system integration tests (with QI providing domain knowledge about individual systems). Devs own tests for their own systems. |
| 📈 **CI reliability** | QS ensures CI signals are trustworthy. Flake reduction, build time optimization, and test health are QS responsibilities. |

---

## A Ticket Is "Done" When...

> ✅ **The system can continuously prove it works.**
>
> - Automated tests cover the acceptance criteria
> - **Not** when QA has time to manually validate it

---

## The Transition: 9–18 Months

This shift will not happen overnight. Jitsu estimates it will take **9–18 months** to fully transition to this model. During this period:

> 🎯 **What Matters Most**
>
> **Momentum and commitment** matter more than immediate perfection. We are looking for steady progress, not instant transformation.

### What Success Looks Like Over Time

| Period | What Success Looks Like |
|--------|-------------------------|
| **📅 Months 1–6** | Devs begin writing more tests. QA shifts toward test design and collaboration. New patterns established on pilot teams. |
| **📅 Months 6–12** | Automation coverage increases significantly. Manual regression testing decreases. QI and QS specializations become clear. |
| **📅 Months 12–18** | Automation is the primary release signal. Large tickets no longer wait on QA. QA spends more time **designing** tests than executing them. |

---

## For Developers During Transition

- **Read acceptance criteria carefully** before coding — AC comes from PO + QI + Tech Lead Three Amigos
- Write tests for your changes, using AI tools where helpful
- **Verify code against the test cases** before pushing the PR for QS review
- Reach out to QA for help identifying edge cases and risk areas
- Treat test authoring as part of "done", not extra work

## For QA During Transition

- Shift time from manual regression toward test design and exploratory testing
- Partner with developers on acceptance criteria and test scenarios
- Begin learning how to use AI tools for test generation and quality improvement
- Identify which specialization (**QI** or **QS**) aligns with your interests and strengths

---

## Jitsu's Commitment

This transition is challenging. Jitsu commits to supporting it through:

| Support | Description |
|---------|-------------|
| 📚 **Training** | Test automation concepts, tools, and patterns for both developers and QA |
| 🔧 **Tooling** | AI-assisted test generation tools that make writing tests nearly effortless |
| 🏗️ **Frameworks** | Jitsu-specific testing frameworks and patterns that lower the barrier to entry |
| 👥 **Mentorship** | Pairing and coaching for those developing new skills |
| ⏰ **Time** | Space to learn and practice without penalty — progress expected, not perfection |

---

## How This Connects to the Ladders

This quality model is reflected in the discipline-specific career ladders:

- **Software Development**: Testing responsibility is woven into level expectations. Developers at all levels are expected to prove their code works.
- **[Quality Improvement Engineering](https://es.gojitsu.com/ecf/qi-engineering.html)**: For those currently on the Manual QA team. Focus on exploratory testing, risk discovery, test design, and encoding domain knowledge through PRs with failing tests.
- **[Quality Systems Engineering](https://es.gojitsu.com/ecf/qs-engineering.html)**: For those currently on the Automation QA team. Focus on AI-assisted test generation, frameworks, CI reliability, and enabling developers to test effortlessly.

Growth in all disciplines increasingly benefits from collaboration on quality. Embracing shared ownership — developers taking pride in well-tested code, QI engineers proactively finding and fixing quality gaps, QS engineers building systems that make quality effortless — creates opportunities for impact and recognition that weren't available in the old model.

---

## 🤝 Our Promise

This shift is **not** about reducing QA or increasing developer burden. It is about:

- Removing structural bottlenecks
- Encoding system knowledge so it compounds
- Making quality scale with velocity
- Enabling AI-assisted development safely

**Quality stops being a role boundary and becomes a shared, compounding capability of the organization.**

---

## Related Documents

- [QI Engineering — Career Ladder](https://es.gojitsu.com/ecf/qi-engineering.html)
- [QS Engineering — Career Ladder](https://es.gojitsu.com/ecf/qs-engineering.html)
- [Probation Goals — Junior + Senior QA](../jitsu-qa-probation-goals.md)

---

*This document is a mirror of the official Jitsu ECF page. For the authoritative version, see [es.gojitsu.com/ecf/quality-ownership.html](https://es.gojitsu.com/ecf/quality-ownership.html).*
