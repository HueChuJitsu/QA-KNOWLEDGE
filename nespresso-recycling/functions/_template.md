---
# ===== IDENTITY =====
feature:                                # kebab-case, e.g. address-editing-in-client-portal
title:                                  # Human-readable, e.g. Address Editing in Client Portal
domain:                                 # From domain-taxonomy.yaml
sub_domain:                             # e.g. address-editing
schema_version: 1.0

# ===== STATE =====
status: active                          # active | deprecated | archived
maturity: evolving                      # experimental | evolving | stable
maintainer:                             # username, e.g. hue.chu

# ===== SEARCH & DISCOVERY =====
keywords:                               # 5-15 items, mix category + phrases
  - 
  - 
  - 

# ===== TAXONOMY =====
user_types:                             # Who uses this feature
  - 
system_touchpoints:                     # Which service/table/component is touched
  - 

# ===== EXTERNAL SOURCES =====
confluence_refs:                        # Related Confluence pages
  - id: 
    title: 
    url: 

figma_refs:                             # Related Figma designs
  - name: 
    url: 
---

# <Feature Title>

> **Scope** — <1-2 sentences defining the boundary. Important in-scope / out-of-scope.>

---

## 1. Actors & Systems

- **<Actor / System>** — <role>
- **`<table name>`** — <purpose>
  - Key fields: `<field1>`, `<field2>`

---

## 2. Business Rules

### 2.1. <Rule Group>

**Allow when ANY condition is true:**

| # | Condition |
|---|---|
| A1 | <condition> |

**Block when ANY condition is true:**

| # | Condition |
|---|---|
| B1 | <condition> |

**Core rule**: <1 memorable, easy-to-recall sentence>

### 2.2. Special Cases
<If any; otherwise: "None currently.">

---

## 3. Workflow

1. <step>
2. <step>
3. <step>

---

## 4. Data Model
<Optional — remove this section if the feature does not touch data>

### Tables

- **`<table>`** — <purpose>
  - `<field>`: <values or description>

### Reference SQL

```sql
-- <Purpose>
SELECT ...
```

---

## 5. Related Behaviors

<How this feature interacts with other features. Ripple effects.>

- **<Behavior area>**: <interaction>. See [<feature>](../path/to/file.md).

---

## 6. Known Issues, Gotchas & Ambiguities

### Currently open issues

- **<Issue>**: <description + impact>

### Gotchas

- **<Gotcha>**: <description>

### Fixed issues (kept for regression coverage)

- **[FIXED — YYYY] <Issue>**: <description>

### Open ambiguities

- **#A1 — <name>**: <question>. Needs confirmation from <PM/Design/Dev>.

---

## 7. References

### Confluence

- [<page title>](<url>)

### Related features

- [<feature title>](../path/to/file.md)

### Glossary

- [<term>](../../glossary/<term>.md)

### Figma

- [<design name>](<url>)
