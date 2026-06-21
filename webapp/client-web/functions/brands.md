# Brands

> Also called **client profiles**. Source: internal note by Evan Robinson.

## 1. Description

**Brands** are a way to **specialize a client's shipment behaviors** — also referred to as **client profiles**. A client (the Jitsu customer) ships on behalf of other companies; each of those companies can be represented as a brand, so notifications, labels, and the API reflect *their* identity rather than the parent client's.

**Actor:** Client (Client Web) — and, in some cases, the brand itself via a restricted client-portal login.

**Why brands? (B2B2B2C)**

- Clients ship on behalf of other companies, so the **parent client's brand is less important than its customers' brands**.
- They want **their brands' name, contact info, etc.** on any notification (email, SMS).
- They want **their brands' logos** on labels.
- They want to **maintain a list of brands** associated with them.
- They want to **pass a brand ID (client profile ID)** in API or CSV shipments.
- They want to **generate reports by brand** (in addition to overall).
- In some cases they want to **create a login** to the client portal for a brand, restricted to that brand's data only.

## 2. Business Flow

A brand overrides a set of the parent client's settings that drive **labeling, notifications, and the API**. When a shipment is submitted with a brand (client profile) ID, the brand's settings apply instead of the parent client's.

### Settings a brand may override

| Setting | Used for |
|---|---|
| Company name | Notifications, labels |
| Company logo | Labels |
| Label template | Labels |
| Recipient page | Notifications / tracking |
| E-mail templates / messages | Email notifications |
| SMS messages | SMS notifications |
| API tokens | API (scoped — see §3) |
| Webhooks | Webhook events (scoped — see §3) |

### How brands are created

```
Brand created
  ├─ "Static"  → in the client portal: a client profile ID + fixed settings
  └─ "Dynamic" → on the fly during shipment submission
```

- **Static brands** — created in the client portal, each with a client profile ID and fixed settings.
- **Dynamic brands** — defined on the fly during shipment submission.
- **All brands** (static *or* dynamic) support customization of the fields listed above.

## 3. Spec / Rules

**API / webhook scoping**

- A **brand API token** supports the shipment APIs **but only for shipments with that client profile ID**. This lets the API be extended to allow querying shipments by sub-client directly.
- A **sub-client (brand) webhook** fires **only for that sub-client's shipment events**.

**Static brands — advanced features**

Static brands additionally *may* support:

- A **filtered view** of the client portal, including the ability for a brand to log into the portal (restricted to that brand's data only).
- **Reporting** with a break-down / filter by brand.
- **Invoicing** by brand.

**Two types of brand**

Every brand has its own row in the `client_profiles` table. The difference is whether the brand also has its own `client_id`:

| Type | Has `client_id`? | Behavior |
|---|---|---|
| **Brand with client_id** | Yes | Acts almost like an independent client. The **only** difference from a real standalone client is that it uses the **parent client's `purchaser_id`** (in `client_profiles`). Can log into the client portal as its own client. |
| **Brand without client_id** | No | **Cannot** log into the client portal as a separate client. The **only** difference is that it has its own client profile (its own row in `client_profiles`). |

```
Parent Client (purchaser_id)
  ├─ Brand WITH client_id      → own client_id, shares parent's purchaser_id → can log into portal
  └─ Brand WITHOUT client_id   → own client_profile only, no client_id       → cannot log into portal
```

**Enabling the Brand feature**

To let a client create their own brands in the Client portal, enable the feature flag:

| Flag | Type | Default | Set in |
|---|---|---|---|
| `use_brand_feature` | String `[true/false]` | `false` | `client_setting.settings` |

> **`use_brand_feature`** — *"Enable brand support. This allows multiple companies' shipments to be handled through a single client."*

Set it to `true` in the client's `client_setting.settings` to turn on brand support for that client.

## 4. QA / Test notes

**Overrides apply correctly**
- [ ] Shipment submitted with a brand ID → notifications (email/SMS) use the **brand's** company name & contact info, not the parent client's.
- [ ] Label renders the **brand's** logo and label template.
- [ ] Recipient/tracking page reflects the brand.
- [ ] Brand with *no* override for a given field → falls back to the parent client's setting (no blank values).

**Feature flag**
- [ ] `use_brand_feature` not set / `false` (default) → the brand UI is hidden; client cannot create brands.
- [ ] Set `use_brand_feature = true` in `client_setting.settings` → client can create/manage brands in the portal.

**Creation**
- [ ] Create a **static** brand in the client portal → gets a client profile ID; settings persist.
- [ ] Create a **dynamic** brand during shipment submission → applied to that shipment.
- [ ] Both static and dynamic brands allow customizing all listed fields.

**Two brand types**
- [ ] Brand **with** client_id → can log into the client portal; shares the parent's `purchaser_id` in `client_profiles`.
- [ ] Brand **without** client_id → cannot log into the portal as a separate client; still has its own `client_profiles` row.

**API / webhook scoping (negative tests)**
- [ ] Brand API token can query/submit shipments **only** for its client profile ID → attempting another brand's / the parent's shipments is rejected.
- [ ] Brand webhook fires **only** for that brand's shipment events → no leakage of other brands' events.
- [ ] Brand ID passed via **CSV** shipment is honored the same as via API.

**Static-brand advanced features**
- [ ] Brand login to the portal sees a **filtered view** — only that brand's data, nothing else.
- [ ] Reporting filtered by brand returns only that brand's shipments; overall report still includes all.
- [ ] Invoicing by brand groups charges to the correct brand.

**Things to watch**
- [ ] Unknown / invalid brand (client profile) ID on a shipment → graceful error, not a crash or silent fallback.
- [ ] Brand login cannot escalate to parent-client or sibling-brand data.
