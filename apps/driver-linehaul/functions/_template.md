# Function: <Function Name>

<!--
Documentation template for a Driver Linehaul function.
Copy this file, rename it to `<function-name>.md`, add a row to the README function
index, then fill in the sections below.

Guidance:
- Keep only the sections that apply. Delete the ones you don't need
  (e.g. a UI-facing screen won't need "Configuration reference"; a backend
  worker won't need "UI elements").
- Prefer tables and short flow diagrams over long prose.
- Quote exact user-facing copy (error messages, placeholders) verbatim in `code`.
- Link the origin ticket and any canonical Confluence spec at the top.
-->

> **App:** Linehaul Driver App · **Origin ticket:** [MOB-XXXX](https://gojitsu.atlassian.net/browse/MOB-XXXX)
> **Confluence:** [<spec title>](<url>) · **Last updated:** <Month YYYY>

## 1. Description

<!-- What this function does, and which actor/population it is for. 3-5 lines max. -->

Key characteristics:

- <bullet — the essentials a reader needs before the details>
- <bullet>

---

## 2. Business flow

<!-- Main steps with entry/exit conditions. Use an ASCII flow when branching matters. -->

```
<step 1>
    ↓
<decision?> → <branch>
    ↓
<outcome>
```

---

## 3. Spec / Rules

<!-- Business rules, validation, states. Break into sub-sections as needed. -->

### Validation / inputs
- **<field>** — <rule>; on failure → `<exact error copy>`.

### States / eligibility
<!-- If behavior depends on a state machine, use a matrix. -->

| State | Condition | Result | Notes |
|---|---|---|---|
| <state> | <condition> | <result> | <notes> |

---

## 4. UI elements & messages

<!-- OPTIONAL — for screens. Delete for backend-only functions. -->

| Element | Default state / behavior |
|---|---|
| `<field/button/link>` | <placeholder / action> |

**Message catalog** (quote copy verbatim):

| Trigger | Message |
|---|---|
| <trigger> | `<exact copy>` |

---

## 5. Configuration reference

<!-- OPTIONAL — for backend/worker features driven by config keys. Delete if N/A. -->

| Key | Type | Storage | Default | Scope | Purpose |
|---|---|---|---|---|---|
| `<config.key>` | <type> | Consul | `<default>` | Global | <what it controls> |

> Note any restart requirement (e.g. keys read in `setup()` → restart the worker).

---

## 6. QA / Test notes

<!-- Happy cases, edge cases, sample data, things to watch when testing. -->

### Happy path
- [ ] <case>

### Edge cases & gotchas

| Scenario | Expected behavior |
|---|---|
| <scenario> | <behavior> |

---

## 7. Discrepancies & open questions

<!-- OPTIONAL — surface spec conflicts / TBDs instead of silently resolving them. -->

| # | Item | Detail | Owner |
|---|---|---|---|
| 1 | <item> | <what needs confirming> | PO / Backend |
