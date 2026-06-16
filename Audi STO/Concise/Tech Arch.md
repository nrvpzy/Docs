# Audi STO Architecture — Concise Business Walkthrough

## 1. What STO Does

**Audi STO — Service Technik Online** is a VIN-centric platform that collects vehicle service information — warnings, recalls, predictive maintenance events, quality/customer information — and makes it available to the right channels:

- myAudi app / portal
- in-vehicle systems
- service portlet / workshops
- internal operations
- GDPR/DSGVO processes

Core purpose:

> **Tell Audi, the customer, the vehicle, and the workshop what is currently relevant for a specific VIN.**

---

## 2. Business Problem

Before STO, customers often learned about recalls or service issues late — through letters, workshop visits, or dashboard warnings.

STO makes communication:

- earlier
- digital
- VIN-specific
- channel-specific
- consistent across systems
- useful for both customers and workshops

---

## 3. High-Level Flow

```text
Upstream Systems
ACDC / Predictive Maintenance / QCI / Recall Sources
        ↓
Kafka Queues
        ↓
STO Dispatcher
        ↓
Internal STO Event
        ↓
STO Supplier
        ↓
Cassandra Vehicle State
        ↓
APIs / myAudi / Vehicle / Workshop / Internal Systems
```

Short version:

> **Events come in → Dispatcher prepares them → Supplier stores and serves them.**

---

## 4. Dispatcher Role

**Dispatcher is the ingestion and routing layer.**

It receives raw events from upstream systems and turns them into clean STO events.

Dispatcher does:

- consumes raw event messages
- validates event data
- identifies the vehicle
- resolves technical IDs/PVIN to VIN using VDS/VIN resolver/VWAC
- normalizes different source formats
- applies configuration rules
- forwards clean events to Supplier
- routes bad events to error flow
- routes reset requests back to source systems
- triggers notification services when required

Memory hook:

> **Dispatcher = receives, understands, identifies VIN, routes.**

---

## 5. Supplier Role

**Supplier is the state and API layer.**

It owns the current vehicle service state and serves it to consumers.

Supplier does:

- receives clean events from Dispatcher
- stores latest VIN-specific state in Cassandra
- avoids stale/duplicate updates
- exposes APIs for:
  - myAudi app
  - vehicle systems
  - workshops/service portlet
  - reset
  - configuration
  - GDPR/DSGVO
- enriches responses using external services
- returns channel-specific data

Memory hook:

> **Supplier = stores current VIN truth and serves it.**

---

## 6. External Services

| Service | Used By | Purpose |
|---|---|---|
| VDS / VIN Resolver / VWAC | Dispatcher | Identify correct vehicle / resolve PVIN to VIN |
| Recall / OEM.IL / Campaigns | Supplier | Fetch recall and campaign details |
| CMS / zFDI | Supplier | Provide localized customer text |
| ORU / Update Status Hub | Supplier | Provide update or campaign execution status |
| CloudIDP / OAuth | Supplier/Dispatcher | Secure token/authentication |
| FNS / MNP | Dispatcher | Send customer notifications |
| Cassandra | Supplier | Store current vehicle state |
| Kafka | Both | Reliable event communication |

---

## 7. Main Business Flows

## New Warning/Event Flow

```text
Vehicle event generated
→ Dispatcher validates and identifies VIN
→ Dispatcher normalizes event
→ Supplier stores latest state
→ Customer/app/workshop can retrieve it
```

---

## Recall Flow

```text
Customer/app requests VIN status
→ Supplier checks stored state
→ Supplier calls recall/campaign service
→ Supplier enriches response
→ Customer sees open recall/campaign
```

---

## Notification Flow

```text
Important event stored by Supplier
→ Notification trigger created
→ Dispatcher calls FNS/MNP
→ Customer receives push notification
```

---

## Reset Flow

```text
Customer resets event in app
→ Supplier validates reset
→ Dispatcher routes reset to source system
→ Source sends updated event
→ Supplier updates state
```

---

## GDPR/DSGVO Flow

```text
Compliance request
→ Supplier finds stored VIN/event data
→ Supplier deletes or reports data
```

---

## 8. Why Dispatcher and Supplier Are Separate

The split gives clear responsibility:

| Dispatcher | Supplier |
|---|---|
| Handles raw incoming events | Owns stored vehicle state |
| Deals with upstream complexity | Serves customer/workshop APIs |
| Stateless/routing-focused | Stateful/API-focused |
| Identifies and normalizes | Stores, enriches, responds |

Business value:

- scalable
- reliable
- easier to change
- failures are isolated
- event bursts do not directly impact APIs

---

## 9. Mock Server Role

The mock server simulates external systems in local/CI environments:

- VDS
- VIN resolver
- Recall/campaigns
- Token service
- FNS
- ORU

It also injects test events into Kafka.

Purpose:

> Allows end-to-end testing without depending on real enterprise systems.

Impact:

- stable CI
- repeatable tests
- faster development
- easier failure simulation

---

## 10. One-Minute Explanation

> Audi STO is a VIN-centric service communication platform. It receives vehicle warnings, recalls, predictive maintenance, and quality events from upstream systems. Dispatcher ingests these raw events, validates them, identifies the correct VIN using vehicle services, normalizes the data, and forwards clean events. Supplier stores the latest vehicle-specific state in Cassandra and exposes APIs to myAudi, in-vehicle systems, workshops, and internal systems. Supplier also enriches responses with recall data, localized CMS text, and update status. Notifications are sent through FNS/MNP, and reset plus GDPR flows are supported. The result is faster, digital, and consistent service communication for Audi customers and partners.

---

## 11. Final Memory Hook

```text
Sources create events.
Dispatcher prepares and routes them.
Supplier stores and serves them.
External services enrich and notify.
Apps, vehicles, workshops consume them.
```

Even shorter:

> **Dispatcher = prepare. Supplier = store and serve. Kafka = transport. Cassandra = state. External services = identity, recall, text, status, notification.**