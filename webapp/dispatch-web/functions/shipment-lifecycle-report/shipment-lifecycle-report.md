# Shipment Lifecycle Report

> **App:** Dispatch Web · **Status:** Draft
> **Shared context:** see [../../README.md](../../README.md)
>
> **Source:** [Confluence — Shipment Lifecycle Report](https://gojitsu.atlassian.net/wiki/spaces/~61e629777ae0dc006a92d2b0/pages/2409988107/Shipment+Lifecycle+Report) (last modified Mar 27, 2026)
> **Data source:** PostgreSQL `staging_apr22` (port 15433) — `shipments` and `stops` tables

## 1. Overview

This report documents the full lifecycle of a shipment and its associated stops (pickup, deliver, return) in the staging system. Data was collected by real-time observation of shipments **69082078**, **69082080**, **69082082**, and **69082083** assigned to assignment **1988773** (driver ID: 1674), querying the `shipments` and `stops` tables in PostgreSQL (`staging_apr22`).

## 2. Key Concepts

### 2.1 Shipment Status Progression

A shipment moves through the following states in order:

```
GEOCODED → ASSIGNED → PICKUP_SUCCEEDED → DROPOFF_SUCCEEDED
                                       ↘ DROPOFF_FAILED → RETURN_SUCCEEDED → RETURNED_TO_CLIENT_SUCCEEDED
                                                                            → RETURNED_TO_CLIENT_DAMAGED
```

### 2.2 Stop Types

Each shipment has up to 3 associated stops:

| Stop Type | Description |
|-----------|-------------|
| `pickupShipment` | Driver picks up the parcel from sender/warehouse |
| `deliverShipment` | Driver delivers the parcel to recipient |
| `returnShipment` | Driver returns the parcel to sender/warehouse — **created only when driver takes action on the App**, not created automatically in parallel |

### 2.3 Stop Status Progression

```
(null/not yet created) → PENDING → EN_ROUTE → READY → SUCCEEDED
                                                     → FAILED (deliverShipment & returnShipment only)
```

- **pickupShipment stop:** only transitions to `SUCCEEDED` (no `FAILED` status).
- **deliverShipment stop:** can transition to `SUCCEEDED` or `FAILED`. In both cases, the driver can trigger a `returnShipment` stop via action on the App.
- **returnShipment stop:** created exclusively by driver action on the App (not created automatically or in parallel with other stops).

## 3. Lifecycle Step-by-Step

| Step | Description |
|------|-------------|
| **Step 1** | Shipment created. Status is `GEOCODED`. No stops exist yet. |
| **Step 2** | Shipment assigned to a driver. Status changes to `ASSIGNED`. Pickup and deliver stops are created with status `PENDING`. |
| **Step 3** | Driver begins the route. Both pickup and deliver stops update to `EN_ROUTE`. |
| **Step 4** | Driver arrives at pickup location. Pickup stop becomes `READY`. Deliver stop remains `EN_ROUTE`. |
| **Step 5** | Driver completes pickup. Pickup stop becomes `SUCCEEDED`. Deliver stop moves to `READY`. Shipment status becomes `PICKUP_SUCCEEDED`. |
| **Step 6** | Driver is en route to deliver. Deliver stop is `EN_ROUTE`. |
| **Step 7** | Driver arrives at delivery location. Deliver stop becomes `READY`. |
| **Step 8** | Delivery attempt fails. Deliver stop becomes `FAILED`. Driver takes action on the App to initiate return — a new `returnShipment` stop is created with status `PENDING`. |
| **Step 9** | Driver is en route to return the shipment. Return stop becomes `EN_ROUTE`. |
| **Step 10** | Driver arrives at return location. Return stop becomes `READY`. |
| **Step 11** | Return is completed. Return stop becomes `SUCCEEDED`. Shipment status becomes `RETURN_SUCCEEDED`. |
| **Step 12** | Final confirmation recorded. Shipment status becomes `RETURNED_TO_CLIENT_SUCCEEDED` (or `RETURNED_TO_CLIENT_DAMAGED` if damaged). |

## 4. Observed Data Tables

### 4.1 Shipments 69082080 & 69082078 (initial group)

| Step | Shipment | Shipment Status | Pickup Stop | Deliver Stop | Return Stop |
|------|----------|-----------------|-------------|--------------|-------------|
| 1 | 69082080 | GEOCODED | — | — | — |
| 1 | 69082078 | GEOCODED | — | — | — |
| 2 | 69082080 | ASSIGNED | PENDING | PENDING | — |
| 2 | 69082078 | ASSIGNED | PENDING | PENDING | — |
| 3 | 69082080 | ASSIGNED | EN_ROUTE | EN_ROUTE | — |
| 3 | 69082078 | ASSIGNED | EN_ROUTE | EN_ROUTE | — |
| 4 | 69082080 | ASSIGNED | READY | EN_ROUTE | — |
| 4 | 69082078 | ASSIGNED | READY | EN_ROUTE | — |
| 5 | 69082080 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 5 | 69082078 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 6 | 69082080 | PICKUP_SUCCEEDED | SUCCEEDED | EN_ROUTE | — |
| 6 | 69082078 | PICKUP_SUCCEEDED | SUCCEEDED | EN_ROUTE | — |
| 7 | 69082080 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 7 | 69082078 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 8 | 69082080 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | PENDING |
| 8 | 69082078 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | PENDING |
| 9 | 69082080 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | EN_ROUTE |
| 9 | 69082078 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | EN_ROUTE |
| 10 | 69082080 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | READY |
| 10 | 69082078 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | READY |
| 11 | 69082080 | RETURN_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |
| 11 | 69082078 | RETURN_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |
| 12 | 69082080 | RETURNED_TO_CLIENT_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |
| 12 | 69082078 | RETURNED_TO_CLIENT_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |

### 4.2 Shipments 69082082 & 69082083 (added to assignment mid-session)

| Step | Shipment | Shipment Status | Pickup Stop | Deliver Stop | Return Stop |
|------|----------|-----------------|-------------|--------------|-------------|
| 2 | 69082082 | ASSIGNED | PENDING | PENDING | — |
| 2 | 69082083 | ASSIGNED | PENDING | PENDING | — |
| 3 | 69082082 | ASSIGNED | EN_ROUTE | EN_ROUTE | — |
| 3 | 69082083 | ASSIGNED | EN_ROUTE | EN_ROUTE | — |
| 4 | 69082082 | ASSIGNED | READY | EN_ROUTE | — |
| 4 | 69082083 | ASSIGNED | READY | EN_ROUTE | — |
| 5 | 69082082 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 5 | 69082083 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 6 | 69082082 | PICKUP_SUCCEEDED | SUCCEEDED | EN_ROUTE | — |
| 6 | 69082083 | PICKUP_SUCCEEDED | SUCCEEDED | EN_ROUTE | — |
| 7 | 69082082 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 7 | 69082083 | PICKUP_SUCCEEDED | SUCCEEDED | READY | — |
| 8 | 69082082 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | PENDING |
| 8 | 69082083 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | PENDING |
| 9 | 69082082 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | EN_ROUTE |
| 9 | 69082083 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | EN_ROUTE |
| 10 | 69082082 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | READY |
| 10 | 69082083 | PICKUP_SUCCEEDED | SUCCEEDED | FAILED | READY |
| 11 | 69082082 | RETURN_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |
| 11 | 69082083 | RETURN_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |
| 12 | 69082082 | RETURNED_TO_CLIENT_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |
| 12 | 69082083 | RETURNED_TO_CLIENT_SUCCEEDED | SUCCEEDED | FAILED | SUCCEEDED |

## 5. Full Lifecycle State Diagram

```
Shipment Created (GEOCODED)
        │
        ▼
Assigned to Driver (ASSIGNED)
  pickupShipment:  PENDING
  deliverShipment: PENDING
        │
        ▼
Driver Starts Route
  pickupShipment:  EN_ROUTE
  deliverShipment: EN_ROUTE
        │
        ▼
Driver at Pickup Location
  pickupShipment:  READY
  deliverShipment: EN_ROUTE
        │
        ▼
Pickup Complete (PICKUP_SUCCEEDED)
  pickupShipment:  SUCCEEDED
  deliverShipment: READY
        │
        ▼
Driver at Delivery Location
  deliverShipment: READY → [attempt delivery]
        │
   ┌────┴────┐
   ▼         ▼
SUCCESS   FAILURE
   │         │
DROPOFF_    deliverShipment: FAILED
SUCCEEDED   │
   │         └─── Driver action on App
   ▼               → returnShipment stop created (PENDING)
  END                    │
                         ▼
                    Driver En Route to Return
                    returnShipment: EN_ROUTE
                         │
                         ▼
                    Driver at Return Location
                    returnShipment: READY
                         │
                         ▼
                    Return Complete (RETURN_SUCCEEDED)
                    returnShipment: SUCCEEDED
                         │
                         ▼
                    RETURNED_TO_CLIENT_SUCCEEDED
                    or RETURNED_TO_CLIENT_DAMAGED
```

## 6. Notes

- **`_deleted` flag:** Both `shipments` and `stops` tables contain a `_deleted` boolean field. Deleted records are soft-deleted and excluded from all queries.
- **Stop creation timing:** `pickupShipment` and `deliverShipment` stops are created when the assignment is made. The `returnShipment` stop is **only created when the driver explicitly takes action on the App** — not automatically or in parallel with other stops.
- **pickupShipment stop:** does not have a `FAILED` status. It only transitions to `SUCCEEDED`.
- **deliverShipment stop:** can transition to `SUCCEEDED` or `FAILED`. In both outcomes, the driver may trigger a return via the App.
- **Reattempt:** In some flows, a new `deliverShipment` stop may be created for a reattempt after a previous one is `FAILED`.
- **Data source:** All data queried from `shipments` and `stops` tables in PostgreSQL database `staging_apr22` on port 15433.
