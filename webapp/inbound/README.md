# Inbound — Project Documentation

> **Project:** Inbound
> **Source-of-truth specs:** Confluence (Engineering space)
> **Last updated:** June 2026

Documentation for the **Inbound** domain. Per-function documentation lives in the
[`functions/`](functions/) directory. Functions that form a group live in their
own sub-folder under `functions/`.

## Function index

| Function | File | Status |
|----------|------|--------|
| Client OTD (OTD by SLA) | [functions/client-otd/otd-by-sla.md](functions/client-otd/otd-by-sla.md) | Production |
| Client OTD — Old Logic (inbound_received_ts) | [functions/client-otd/old-otd-inbound-received-ts.md](functions/client-otd/old-otd-inbound-received-ts.md) | Deprecated |

> To add a new function: copy [`functions/_template.md`](functions/_template.md) to
> `functions/<function-name>.md`, then add a row to the table above.

---

## 1. Project Overview

<!-- High-level description of the Inbound domain, its actors, and scope. -->

## 2. Open Questions

<!-- Track unresolved spec questions here. -->

## 3. Contributors

<!-- Names / roles of people maintaining this documentation. -->
