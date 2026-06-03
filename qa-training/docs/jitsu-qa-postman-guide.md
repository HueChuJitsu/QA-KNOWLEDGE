# Jitsu QA — Postman Guide

**JITSU QA TEAM**

**Postman Setup & Usage Guide**

*How to install, configure, and use the QA Master Collection*

**📋 INSIDE THIS DOC**

- ✓ Install Postman + import QA Master Collection
- ✓ Configure environments: Staging / Beta / Production
- ✓ Authentication: tokens, login, refresh
- ✓ Common QA use cases (submit shipment, simulate FCRA, scan...)
- ✓ Pre-request scripts & chaining requests
- ✓ Troubleshooting common errors

*Version 1 — May 2026*

---

## Table of Contents

1. [Install Postman & Login](#1-install-postman--login)
2. [Import the QA Master Collection](#2-import-the-qa-master-collection)
3. [Configure Environments (Staging / Beta / Prod)](#3-configure-environments)
4. [Authentication & Token Management](#4-authentication--token-management)
5. [Common QA Use Cases](#5-common-qa-use-cases)
6. [Pre-request Scripts & Chaining](#6-pre-request-scripts--chaining)
7. [Tips & Troubleshooting](#7-tips--troubleshooting)

---

## 1. Install Postman & Login

### Download Postman

- Go to https://www.postman.com/downloads/
- Download the desktop app for your OS (macOS / Windows / Linux).
- Install it normally.

### Create Postman account

- Open Postman.
- Sign up with your Jitsu email (@jitsu / @gojitsu / @axlehire) — recommended so the QA Lead can share the workspace with you.
- Verify your email.

> **💡 TIP — Use the Jitsu workspace**
> If the team uses a shared Postman workspace, ask the QA Lead to invite you. This ensures everyone has the latest collection version. Otherwise, you'll need to manually re-import when there are updates.

---

## 2. Import the QA Master Collection

The Jitsu QA team maintains a master Postman collection with all the API requests you'll need. Get it imported on Day 1.

### Get the collection

Ask your QA Lead or Buddy for the master collection. It usually comes in one of these formats:

- Postman workspace invite (best option — auto-syncs future updates)
- `.json` file (export from team workspace)
- Postman shared link

### Import the collection

#### Option A — From a .json file

- Open the Postman desktop app.
- Click 'Import' (top-left, next to 'New').
- Drag the `.json` file in, OR click 'Choose Files' and select it.
- Click 'Import'.
- The collection appears in the left sidebar — usually named 'Jitsu QA Master' or similar.

#### Option B — From a Postman workspace invite

- Accept the workspace invite from your email.
- Postman desktop will sync the workspace automatically.
- Switch to that workspace via the workspace dropdown at the top of Postman.

### Verify import

- Expand the collection in the left sidebar.
- You should see folders like: `[Auth]` / `[Client]` / `[Driver]` / `[Routing]` / `[Inbound]` / `[Reg]` / `[Payment]`...
- Each folder has individual API requests.

---

## 3. Configure Environments

Postman environments let you switch between Staging / Beta / Production without changing URLs everywhere. The QA Master Collection comes with environment templates — you need to import + configure them.

### Import environment files

Usually 3 files come with the master collection:

- `Jitsu_Staging.postman_environment.json`
- `Jitsu_Beta.postman_environment.json`
- `Jitsu_Production.postman_environment.json`

- Click 'Import' in Postman.
- Drop in all 3 environment files (or import one at a time).
- They appear in the 'Environments' tab in the left sidebar.

### Switch environment

- Use the environment dropdown at the top-right of Postman.
- Choose 'Jitsu Staging' for most QA work.
- Switch to 'Jitsu Beta' for pre-prod verification.
- Switch to 'Jitsu Production' ONLY when explicitly needed — follow Production rules (Training Doc Section 10).

### Common environment variables

| Variable | Purpose |
| --- | --- |
| **dataorch_url** | Base URL for the Data Orchestrator service (internal). Used by many APIs. |
| **routing_app_url** | Base URL for the Routing service. |
| **inbound_api_url** | Base URL for the Inbound service (scan, audit). |
| **admin_token** | Token for admin operations (e.g. cloning shipments). Refreshed when expired. |
| **client_token** | Public token of a test client. Used for submitting shipments via API. |
| **driver_token** | Access token of a test driver. Used for driver-side APIs (book, deliver). |
| **solutionId, assignmentId, shipmentId** | Set dynamically by chained requests (auto-populated, see Section 6). |

> **💡 TIP — Edit environment values**
> Click the 'eye' icon next to the environment dropdown to view current values. Click 'Edit' to update tokens or IDs. Always update the test `client_token` after creating a new test client.

---

## 4. Authentication & Token Management

### How Jitsu APIs authenticate

Most Jitsu APIs use Bearer tokens in the Authorization header:

```
Authorization: Bearer <token>
```

Tokens differ by role:

- **Admin token** — for admin operations (e.g. clone shipment from Prod)
- **Client token (public)** — for client-side operations (submit shipment)
- **Driver token** — for driver-side operations (book, deliver)

### Get tokens via login API

In the QA Master Collection, look for the `[Oauth]` or `[Auth]` folder. Common requests:

#### [Oauth] Admin Login

*Returns admin_token. Used for admin operations.*

```
POST {{oauth_url}}/login
Body: {
  "email": "admin@jitsu.com",
  "password": "..."
}
```

#### [Oauth] Client Login

*Returns client_token (public). Used for the submit shipment API.*

#### [Oauth] Driver Login

*Returns driver_token. Used after signing up a test driver via the Driver App.*

### Auto-save tokens to environment

Most login requests in the collection have a 'Tests' script that automatically saves the returned token to the environment. Example:

```javascript
// In Tests tab of the Login request
var jsonData = pm.response.json();
pm.environment.set("admin_token", jsonData.access_token);
```

So after running Login → all subsequent requests using `{{admin_token}}` will work automatically.

> **⚠️ Token expires every few hours**
> If you start getting 401 Unauthorized errors, run the Login request again to refresh the token. Don't manually edit tokens — use the Login request.

---

## 5. Common QA Use Cases

The most frequent Postman tasks you'll do as a QA at Jitsu. All reference the QA Master Collection.

### 5.1 Submit a single shipment

- **Folder:** `[Client]` or `[Public API]`
- **Request:** 'Submit Shipment'
- **Method:** POST
- **Auth:** uses `{{client_token}}` — make sure your test client's token is in the environment
- **Body:** JSON with recipient_name, address, weight, dimensions, etc.

### 5.2 Clone shipment from Production

- **Folder:** `[Admin]` or `[Maintenance]`
- **Request:** 'Clone shipment for other client'
- **Auth:** uses `{{admin_token}}` (Production environment)
- **Input:** list of shipment_ids to clone
- **Output:** CSV file downloaded — upload to Staging Client web
- **Reference:** Training Document Section 11.3

### 5.3 Simulate FCRA approval (Driver BG check)

- **Folder:** `[Reg]` (Registration)
- **Request:** '7.[Reg]Simulate fcra approval'
- **Auth:** uses `{{driver_token}}`
- **Use:** after signing up a test driver via the Driver App, run this to skip the real BG check
- ⚠️ Staging only — never run on Production
- **Reference:** Training Document Section 11.5

### 5.4 Simulate Turn status (BG check state)

- **Folder:** `[Reg]` or `[Turn]`
- **Request:** 'Outcome received'
- **Use:** set the BG check state manually (passed, failed, in_progress)
- **Updates MongoDB collection:** `driver_registration_record`
- **Reference:** Training Document Section 11.5 + Turn API docs

### 5.5 Scan all shipments of a solution (Inbound automation)

- **Folder:** 'scan all shipments in solution' (custom)
- **Run via:** right-click folder → Run Collection
- Chains 3 requests automatically: get assignments → get shipments → scan each
- **Reference:** Training Document Section 11.9

### 5.6 Trigger weekly payment worker

- Done via the Dashboard Schedules UI, not direct Postman
- **URL:** https://dashboard.staging.gojitsu.com/schedules
- **Trigger:** driver_payment worker with custom body
- **Reference:** Training Document Section 11.12

---

## 6. Pre-request Scripts & Chaining

Some QA workflows require multiple API calls in sequence (e.g. scan all shipments in a solution). Postman supports chaining via pre-request and test scripts.

### Pre-request Script — runs BEFORE the request

Use for: setting variables, generating dynamic values, popping items off a queue.

```javascript
// Example: get next shipment from a list
var list = JSON.parse(pm.environment.get("shipmentIds"));
pm.environment.set("shipmentId", list.shift());
pm.environment.set("shipmentIds", JSON.stringify(list));
```

### Tests Script — runs AFTER the request

Use for: validating response, saving values to environment, chaining to the next request.

```javascript
// Example: save returned token + chain to next request
pm.test("Status is 200", function () {
  pm.response.to.have.status(200);
});
var jsonData = pm.response.json();
pm.environment.set("admin_token", jsonData.access_token);

// Chain: if more items left, run the same request again
var remaining = JSON.parse(pm.environment.get("shipmentIds"));
if (remaining.length > 0) {
  postman.setNextRequest("scanShipment");
}
```

### Run a folder as collection

- Right-click a folder → 'Run folder'
- The Collection Runner opens.
- Click 'Run' — Postman executes requests in order, respecting `setNextRequest()` chains.
- Output shows pass/fail for each request.

> **💡 Real example**
> The 'Scan all shipments' workflow (Training Doc 11.9) uses 3 chained requests via `setNextRequest()` to scan dozens of shipments automatically. Study that example — it's the template for any batch operation.

---

## 7. Tips & Troubleshooting

### Common Errors

| Error | Likely Cause | Fix |
| --- | --- | --- |
| **401 Unauthorized** | Token expired or missing | Run the Login request to refresh the token. Verify the Authorization header has `{{token_name}}` populated. |
| **403 Forbidden** | Token valid but no permission | Using wrong token type (e.g. client_token for an admin API). Check which token the API requires. |
| **404 Not Found** | URL incorrect or resource doesn't exist | Verify the environment is set correctly (Staging vs Beta vs Prod). Check the `{{shipmentId}}` or `{{driverId}}` variable. |
| **400 Bad Request** | Invalid request body | Check the Body tab — is the JSON format valid? Required fields present? Check API docs for the expected schema. |
| **500 Server Error** | Server-side issue | Check Datadog logs around the time of the request. May be a bug — report to Dev with the request payload + timestamp. |
| **Network error** | VPN / StrongDM issue | Check if the internal URL requires the StrongDM proxy (e.g. `*.gojitsu.sdm.network`). Make sure SDM is connected. |

### Tips for QA

- Save useful test payloads as 'Examples' in the request — easy to recall later.
- Use Postman's 'History' tab to see your past requests.
- When debugging, click 'Visualize' to see the formatted response (if defined).
- Document edge cases you discover in the request 'Description' tab — helps future QA.
- Never commit Production tokens to shared files (1Password is for shared secrets).

### When to ask for help

- Collection seems outdated → ask QA Lead for the latest version
- New API not in the collection → ask Dev to share the Postman request, then save to your collection
- Tokens don't work even after a fresh login → check with QA Lead, may be an account permission issue

> **📚 Cross-reference**
> This Postman guide complements the Jitsu QA Training Document (Section 11). All E2E steps that use Postman reference this collection. Always read the specific section in the Training Doc for context, then come back here for HOW to use Postman.

---

**End of Postman Guide 🚀**

*When in doubt, check the QA Master Collection's request descriptions first.*
