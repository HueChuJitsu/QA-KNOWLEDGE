# Jitsu Context

> Foundational domain document for QA training. Describes Jitsu — a last-mile delivery company — covering terminology, organization, the shipment lifecycle, and the underlying apps and technology.

## Overview

Jitsu is a last-mile delivery company and technology platform. Its customers are e-commerce retailers looking for alternatives to USPS, national carriers (UPS, FedEx), and regional carriers (OnTrac).

- **Asset-light**: owns no delivery vehicles, leases facilities, and rents "mobile hubs" month-to-month → lower fixed costs and the flexibility to scale up or down.
- Achieved **SOC 2 Type II** in April 2024 (audited by Sensiba LLP).

## Key Terminology

| Term | Definition |
|---|---|
| **Shipment** | A package Jitsu receives from a client and delivers to a recipient (customer). Each shipment has a geocoded delivery address. |
| **Route** (a.k.a. "assignment") | An ordered sequence of stops assigned to one driver: ≥1 pickup stop (load at a facility) → delivery stops. |
| **Zone** | A contiguous geographic area with high delivery density. Used for routing and driver booking. |
| **Stem miles** | The distance from the facility to the first delivery stop and from the last stop back. Minimizing it is a primary optimization goal. |
| **OTD** (on-time delivery) | % of shipments delivered within the committed window. One of the core quality metrics. |
| **Courier** | Refers only to DSP or 3P drivers. **Never** refers to IC drivers (Jitsu-specific usage). |
| **SLA** | Delivery speed commitment: same-day, next-day, standard (deferred). |
| **Tour cost** | The base pay of a route before any bonus is added. |

## Markets and Facilities

Jitsu operates in ~29 regions/markets across the US: IND, HOU, DFW, AUS, STL, CHI, MKE, DTW, SFO, LAX, SMF, SDLAX, LAS, SEA, HPN, HVN, CVG, JFK, BWI, EWR, PHL, PIT, IAD, RIC, PHX, ORF, CMH, PDX, CLE.

Each facility can serve one or more functions:

- **Sortation center**: receives shipments ("inbound"), scans + labels them, and sorts them into a route position (locally-bound) or a container bound for another market (cross-dock).
- **Delivery station**: where sorted routes are staged and deployed to drivers ("outbound").
- **Cross-dock center**: receives shipments bound for multiple distant markets and sorts them into containers for middle-mile transport.

**Mobile hub**: a lightweight delivery station, rented month-to-month, placed in satellite areas to reduce stem miles. The decision to open one can be made dynamically — as late as the day before — based on density forecasts.

## Workforce

### Driver Fleet
Jitsu **does not hire drivers directly**. The fleet is a mix of:

- **IC (Independent Contractor)**: individual drivers, booked through the driver app, typically delivering within a specific zone. **Never called couriers.**
- **DSP (Delivery Service Provider)**: contracted delivery companies, paid by package count/type/mileage, not bound to a zone.
- **3P (Third-party gig)**: platforms such as WorkWhile, InstaWork, Wonolo — used when IC/DSP capacity is insufficient (last resort), not bound to a zone.

> DSP + 3P = "couriers". Jitsu typically launches a new market with DSPs, then gradually shifts toward ICs.

### Facility Staff
Mostly contingent workers hired temporarily from labor platforms (fitting the asset-light model).

## Client Lifecycle

- **Sales**: SDRs scout accounts → engage Account Executives.
- **Pricing & Network Design**: pricing + linehaul + sales build the network design (the cheapest way to get packages to Jitsu — "first mile"). First mile is one of the biggest challenges because Jitsu lacks a nationwide induction-point network.
- **Implementation**: connects the client's systems to Jitsu. REST API at https://docs.gojitsu.com/ (with near-real-time webhooks), supports SFTP inbound/outbound, integrations with AfterShip, Narvar, Intelligent Audit, etc.
  - Contract pricing is coded into a rate table via a **pricing DSL** (a spreadsheet-style formula language, e.g. `baseRate + (boxes * perBoxRate) + $IF(isRemote, remoteSurcharge, 0)`). Each price calculation produces a full trace for audit/dispute.
- **Client Success**: manages the account post go-live, handles escalations, and rolls out new regions.

## Operations Organization

- **Linehaul**: first-mile freight coordinated by Jitsu + middle-mile (moving packages between Jitsu facilities).
- **Facilities**: operates sortation/delivery/cross-dock centers and supervises temp workers.
- **Transportation**: driver services, planners, coordinators, dispatchers.
  - **Driver Services**: recruits ICs, background checks (Turn.io, FCRA-compliant), planning, discipline. A 5-level suspension system: complete blackout → delayed access → reduced capacity → minor penalty. Each level is clearly explained to the driver.
  - **Transportation Planners**: (1) find/negotiate with DSPs, (2) use the routing app to build optimal routes.
  - **Transportation Coordinators**: proactively encourage drivers to accept routes.
  - **Dispatchers**: reactively handle driver incidents, using the Jitsu Dispatch app.

## Shipment Lifecycle

### First Mile
Shipments arrive at a facility via the first mile (coordinated by Jitsu Linehaul or arranged by the client/freight partner).

### Inbound and Sortation
Trucks arrive at the sortation center → unload onto conveyors → scan + apply a **sort label**:
- **Route position** (locally-bound → "final sort"): delivered from this market.
- **Region code** (cross-dock → "regional sort"): bound for another market.

Final-sorted: large packages → pallet (a grid where each position = one route); small packages → bin. Cross-docked → container/loading area by destination market.

The routing engine has already created routes before physical sorting. **Sprinkling**: inserting unexpectedly-arriving shipments into the optimal position on a route that still has capacity — to the sort worker it looks like a pre-routed shipment. After sorting is complete ("locked"), Facilities prepares cross-dock and wraps pallets to move them to outbound staging.

### Cross-Docking
Receives packages bound for multiple distant markets → applies a region code, sorts them into containers → Linehaul arranges middle-mile transport.

### Sortation Layout
The physical arrangement of pallets/bins on the sort floor. Generated automatically by a constraint-based layout optimizer, or manually by the sort lead. Optimization strategies:
- Farthest route / earliest deploy (e.g. mobile hub) → near the dock door, at the edge of the grid.
- Group by geography so drivers can pick up several nearby routes.
- Oversized route → automatically split across multiple positions.
- Routes bound for a mobile hub via truck → stack to optimize space (14ft van, 26ft/53ft truck).
- Routes created mid-cycle → automatically added to the layout + advance booking session.

### Multi-Cycle Operations (MCO)
A second sort cycle can begin while cycle-1 routes are still outbound — cycle-1 pallets/bins are "extracted" and staged to free up positions.

## Routing

Building the set of routes for one market on one day: forecast shipments, determine destinations, and compute cost-optimal routes for the mixed fleet.

### Predicting Inbound Volume
- Some clients (subscription, meal-kit) provide an accurate manifest → the "backbone", but these are the minority.
- Most e-commerce manifests are inaccurate → Jitsu uses a predictive model (historic trends, first-mile origin, lag time, per-client patterns).
- Geographic distribution is also needed → uses historical delivery data.
- **Guessed shipments**: fake shipments at the position/quantity the model predicts. When a real shipment is sorted, the system replaces the nearest guess with the real package → keeping the route count stable.

### Service Levels and Deferred Inventory
Day-definite (same-day/next-day) = the backbone (hard-constrained by day). Deferred (standard) is flexible, inserted wherever it improves route density. Planners view "inventory" in the Inbound app.

### Route Creation
Tiered by vehicle size:
- **Super routes**: largest vehicles.
- **Minivan/large routes**: SUVs, wagons, etc.
- **DSP routes**: volume supplied to DSPs by minimum weekly volume.
- **Zone routes**: standard ICs within a zone.
- **LITE routes**: small routes, small vehicles, near the delivery station.

The engine clusters shipments (proprietary) → builds an initial solution (prioritizing hard-to-place shipments first) → iteratively optimizes via **"ruin and recreate"**. After the planner selects a solution, the advance booking session updates the real routes, issues additional booking tickets; drivers held back due to low ratings are opened up for the session.

### Sprinkling and Redelivery
- **Sprinkle**: insert an unexpected shipment into a route that still has capacity.
- **"No routes"**: when no sprinkle is possible → the redelivery app uses regret-based optimization to find the globally best position.

### Mobile Hub Routing
The engine designates clustered routes around a mobile hub; Linehaul arranges the middle-mile truck by volume. A dynamic decision, made as late as the day before.

## Driver Booking

Escalation: advance booking → direct booking → 3P.

- **Advance Booking**: for good-standing ICs by driver rating (on-time pickup, OTD, completion %, delivery-within-window). Forecasts ticket count from the prior week, opens by zone, holds back a portion for day-of. Drivers who cancel late are penalized → they book later.
- **Direct Booking**: when advance is insufficient, before resorting to 3P. Shows exact price + position (unlike ticket booking, which shows only zone/pay range/shipment count range). A bonus can be added on top of the tour cost.
- **Third-Party Escalation**: 3P gig is the last resort when IC + DSP capacity is insufficient.

### Pickup Slots and Yard Management
Each delivery station has pickup slots (~30 min), 1-to-1 with a ticket/route, applied to IC/DSP/3P. There is a parking slot map; drivers arriving early → parking waitlist. Drivers arriving off-slot → lower rating, offered a later opportunity.

## Outbound

Begins once a driver has booked into a delivery station.
- **IC**: checks in by ID → assigned a parking spot → selects a route via **"hybrid booking"** (a set of staged routes chosen by Outbound — a hybrid of driver's choice + Jitsu's choice). Selecting a route attaches the ticket to it.
- **DSP/3P**: arrive with a pre-assigned route.

Goal: maximize flow from arrival → departure. **Driver wait time** is a key KPI (unpaid waiting time is a major source of frustration). The outbound clerk brings the pallet + bin bag → the driver scans + loads, reviews, accepts, departs → Dispatch monitors.

### Apps in Outbound
- **Jitsu Drive** (driver app): booking, pickup, delivery, return, payment.
- **Jitsu Outbound** (outbound app) + Outbound Dashboard web: track drivers, check-in, stage routes, handle exceptions.

## Delivery

### Geofencing
- Driver check-in is geofenced (parking + claiming a route only works when near the yard).
- Delivery is also geofenced (delivered at the right location). Delivering outside the geofence requires correcting the position or contacting Dispatch.
- The geocode/geofence DB has 9 providers in a fallback chain, classifies addresses (residential, military, school, MDU, etc.), and caches for up to 1 year.

### Proof of Delivery (POD)
A POD photo is required for **every shipment**. The photo must show the package at the delivery location + a clearly visible house/unit number.
- **YOLO11** package detection on-device via ONNX Runtime (real-time, no network needed).
- A parallel OCR pipeline (**DBNet + CRNN**) extracts address/unit text.
- Image quality is assessed on-device → immediate retake feedback.
- Rule validation is configured per-client via an **ANTLR DSL**.
- Cloud ML (Google Cloud Vision, Gemini VLM) for complex batch analysis.

### Route Monitoring and Dispatch
Tracks progress; delayed/stalled → escalate: push → SMS → phone call from Dispatch. Drivers must finish within their time window; late → lower rating. The Dispatch app allows "rescuing" a route (splitting off part of it, assigning another driver at a rescue location) to protect OTD.

### Fraud Detection
5 batch algorithms: cost anomaly, impossible travel, completion time, dwell time, POD daylight validation.
6 real-time detectors: missort detection, GPS signal monitoring, unauthorized departure, new-driver monitoring (+ more). Thresholds are configured per region/tier via **Drools**. Automatic triage: high-risk → Slack alert for manual review; low-risk → auto-resolve with metrics.

## Customer Experience

- **Tracking**: a tracking page showing milestones, status, and live ETA (refreshed every 5 minutes based on driver location after pickup).
- **Manage Delivery**: the recipient confirms/updates address, instructions, geocode pin, access code, delivery photo. Shown to the driver when it is the next stop; changes are reflected in the driver app in real time.
- **SMS**: notifies milestones, requests confirmation of missing/ambiguous info; also sends info to drivers (booking notifications).
- **Driver-Recipient Communication**: text/call via an anonymized proxy to resolve delivery issues.

## Platform Apps and Services

### Web Apps
- **Dispatch**: monitor the day's routes, review historical routes, plan future routes; filterable map; click a route → shipments; click a shipment → POD, event history, metadata.
- **Inbound**: shows sort layout/warehouse/day; monitors PPH; build layouts manually when needed.
- **Small Sort**: like Inbound but for bins (small parcels).
- **Outbound Dashboard**: track drivers arriving at the delivery station.
- **Routing**: planners create routing problems + compute solutions.
- **Admin Dashboard**: configure regions, clients, SLAs, warehouses, booking params, DSPs, driver programs, etc.
- **DSP Portal**: DSPs manage their routes + fleet.
- **Recipient Portal**: the customer tracking page + Manage Delivery.
- **Payment**: review IC/DSP payments, manage abnormal routes, configure payment formulas.
- **Client Portal**: shippers track shipments + delivery history.

### Mobile Apps
- **Jitsu Drive** (driver app): sign-up, background check, set route preferences, book/pickup/deliver routes, payment.
- **Jitsu Outbound** (outbound app): manage parking spots, track drivers, "check out" routes.
- **Jitsu Warehouse**: sort workers receive containers, pre-sort (first scan), large/small sort, audit sort positions.

### API Services
- **Public API**: shippers/platforms (Shippo, EasyPost) get rate quotes, submit shipments, generate labels, track, cancel, update.
- **Driver API**: the backend for Jitsu Drive.

## Platform Technology

### Tech Stack
- **Backend**: Java (Dropwizard, Quarkus) primarily; Python (Flask, Celery) for ML/data; Kotlin (Ktor) for newer services. Service-to-service: **gRPC**. Business rules: **Drools**. Pricing DSL + POD DSL: **ANTLR**.
- **Mobile**: Flutter/Dart.
- **Web**: React.js (TypeScript in newer apps), some Vue.js.
- **Data**: PostgreSQL (primary relational), MongoDB (document), Cassandra (event/time-series), Redis (cache, locks, rating ranking), Elasticsearch (search), BigQuery (analytics). Object storage: S3 + Google Cloud Storage.
- **ML/AI**: YOLO v11 + ONNX Runtime (on-device), Google Cloud Vision + Gemini VLM (cloud), XGBoost + SARIMA (forecasting), Vertex AI RAG (knowledge retrieval).
- **Routing/Mapping**: Google OR-Tools, OptaPlanner, jsprit (forked VRP), GraphHopper (traffic-aware), Uber H3 (geospatial).
- **Infra**: GCP, Consul (service discovery), Datadog (observability).

### Event-Driven Architecture
>1 billion events/year, 392 event types, 213 worker processes. Routed via **RabbitMQ**, persisted to **Cassandra**.
- High-volume: shipper webhooks (265M/year), delivery status (218M), package scans (177M), driver location (152M), POD photos (90M).
- One event ("package delivered") fans out to ≥5 workflows: customer notification, driver rating, shipper webhook, fraud check, billing.
- Full audit trail: timestamp, actor, payload, causation chain.

### Offline-First Driver App
- A local event queue in **Hive**, auto-uploaded when connectivity is available.
- Caches full route data → can operate offline.
- POD photos saved locally, background sync, handling interruptions/duplicates/failures.
- Persists app state → recovers after crash/background/restart.
- Anti-tampering: root/jailbreak detection, fake location detection, NTP-based time validation (true-time lib) to prevent clock manipulation.
