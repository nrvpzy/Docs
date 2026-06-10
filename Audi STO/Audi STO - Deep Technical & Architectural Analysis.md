# Audi STO — Deep Technical & Architectural Analysis

## Focus: Supplier, Dispatcher, Mock Server, Common Libraries, Cassandra, OAuth, Integration Tests

### Framed as a Java / Spring Boot Architecture

---

## 1. System Technical Overview

Audi STO is best understood as a **distributed event-processing and API-serving platform**.

At the center of the architecture are two main services:

1. **Dispatcher Service**  
   The ingestion and routing service.

2. **Supplier Service**  
   The state management and API-serving service.

Around these two services are several supporting modules:

- **Mock Server / mockhttp4s** — local and CI simulation of external enterprise systems.
- **STO Commons Library** — shared Kafka, HTTP, config, CMS, telemetry, model, and utility code.
- **Cassandra Commons** — Cassandra connector, data models, schema migration tooling.
- **OAuth Validator Library** — token validation and authentication filters.
- **09 Commons** — shared clients/services for VDS, VWAC, zFDI, token handling, retry, producer utilities.
- **Integration Tests** — end-to-end tests validating real system flows using containerized dependencies.

A high-level technical architecture looks like this:

```text
External Event Producers
ACDC / Predictive / QCI / E3 / Recall-related feeds
        |
        | Kafka raw event topics
        v
+--------------------------+
| Dispatcher Service       |
|--------------------------|
| - Kafka consumers        |
| - Event parsing          |
| - Validation             |
| - VIN/PVIN resolution    |
| - Config lookup          |
| - Event normalization    |
| - Notification routing   |
| - Reset routing          |
+--------------------------+
        |
        | Kafka internal topic: normalized service event
        v
+--------------------------+
| Supplier Service         |
|--------------------------|
| - Kafka consumer         |
| - Idempotency logic      |
| - Cassandra writes       |
| - REST APIs              |
| - Channel filtering      |
| - Recall enrichment      |
| - CMS/zFDI localization  |
| - ORU enrichment         |
| - DSGVO deletion         |
+--------------------------+
        |
        v
+--------------------------+
| Cassandra                |
|--------------------------|
| Current VIN event state  |
| QCI state                |
| OCC/Recall cache         |
| TTL-based lifecycle      |
+--------------------------+
        |
        v
Consumers:
myAudi App / Portal / Vehicle / Workshops / Partner APIs / Admin / Compliance
```

The key design philosophy is:

> **Dispatcher handles input complexity. Supplier handles state correctness and consumer-facing APIs.**

---

# 2. Core Architectural Pattern

The architecture is a combination of:

- **Event-driven architecture**
- **CQRS-style separation**
- **Stateful read model**
- **Asynchronous command processing**
- **Config-driven business behavior**
- **Resilient external-service enrichment**

It is not pure CQRS in the strict event-sourcing sense, because Cassandra stores the **current state**, not a complete event log. But conceptually:

- Kafka events are the write-side input.
- Cassandra is the materialized current-state read model.
- Supplier APIs expose that read model to consumers.

So you can explain it as:

> We used Kafka to decouple event ingestion from state persistence and API serving. Dispatcher normalized raw events into internal business events. Supplier consumed those events and maintained a VIN-keyed Cassandra read model that was optimized for fast customer, vehicle, and workshop queries.

---

# 3. Dispatcher Service — Technical Deep Dive

## 3.1 Role of Dispatcher

The Dispatcher is the **event ingestion gateway**.

It receives raw events from multiple upstream systems. These systems may differ in:

- message format
- event schema
- vehicle identifier type
- brand representation
- regional context
- event semantics
- reset behavior
- notification behavior

The Dispatcher hides this upstream complexity from the rest of STO.

In Spring Boot terms, the Dispatcher would be implemented using:

- `@KafkaListener` consumers
- deserializer components
- validation services
- normalization services
- vehicle identity resolution service
- config cache service
- `KafkaTemplate` producers
- WebClient-based external clients
- retry/backoff components
- health indicators

---

## 3.2 Dispatcher Main Responsibilities

The Dispatcher has four major responsibilities:

```text
1. Ingest raw upstream events
2. Normalize and enrich them
3. Publish internal STO events
4. Route side effects: notifications and resets
```

---

## 3.3 Dispatcher Event Processing Pipeline

A typical Dispatcher flow:

```text
Raw upstream Kafka message
        |
        v
Deserialize payload
        |
        v
Validate required fields
        |
        v
Normalize source-specific data
        |
        v
Resolve vehicle identity
        |
        v
Apply event configuration
        |
        v
Build normalized internal service event
        |
        v
Publish to internal Kafka topic
        |
        v
Commit offset / acknowledge processing
```

In Spring Boot terms:

```text
@KafkaListener
   -> RawEventDeserializer
   -> EventValidationService
   -> EventNormalizationService
   -> VehicleIdentityResolutionService
   -> EventRoutingConfigService
   -> InternalEventProducer
```

---

## 3.4 Raw Event Ingestion

Upstream systems publish messages to Kafka. Dispatcher consumes these messages.

The upstream event families include:

- ACDC vehicle events
- Predictive maintenance events
- QCI / quality customer information events
- E3 regional events
- possibly recall-style or service-campaign-related signals

These events are not directly stored in Cassandra. They first need to be converted into STO’s internal model.

Technical problems at this stage:

1. **Payload format variation**  
   Different sources may send different envelope structures.

2. **Schema/version compatibility**  
   Older and newer event formats may coexist.

3. **Bad messages**  
   Some payloads may be malformed or incomplete.

4. **Different source semantics**  
   A status or field from one system may not mean the same thing in another.

Dispatcher solves this by using a parsing and normalization layer.

---

## 3.5 Event Deserialization and Validation

Before any business processing, Dispatcher must deserialize the message.

Possible outcomes:

| Outcome                   | Handling                                     |
| ------------------------- | -------------------------------------------- |
| Payload can be parsed     | Continue processing                          |
| Payload is malformed      | Route to error flow / DLQ                    |
| Required fields missing   | Route to error flow / DLQ                    |
| Unsupported event version | Route to error flow or compatibility handler |

In a Spring Boot implementation, this would typically look like:

```text
Kafka ConsumerRecord
        |
        v
try deserialize
        |
        +-- success -> domain object
        |
        +-- failure -> publish to DLQ, acknowledge original offset
```

Important principle:

> A malformed event should not poison the whole Kafka partition forever.

So bad messages are treated differently from infrastructure failures.

- **Bad business/payload message** → DLQ and commit
- **Temporary infrastructure failure** → do not commit; retry later

---

## 3.6 Event Normalization

After deserialization, Dispatcher normalizes the raw event.

Normalization may include:

- Mapping brand codes into standard brand values.
- Mapping source event types into STO model IDs.
- Extracting additional information fields.
- Standardizing status values such as active/resolved/passive.
- Extracting text references, picture references, odometer, recall action codes, reset codes.
- Attaching trace context.
- Building a common internal representation independent of the source system.

Example:

```text
Raw source event:
brand = "AU"
vehicleId = "PVIN-123"
eventCode = "W042"
status = "NOK"
txtId = "TEXT_123"

Normalized STO event:
brand = "AUDI"
vin = resolved later
modelId = "W042"
status = "ACTIVE/NOK"
textReference = "TEXT_123"
sourceSystem = "ACDC"
```

The goal is:

> Supplier should not need to understand every upstream dialect.

---

## 3.7 VIN / PVIN Resolution

This is one of the most technically important Dispatcher responsibilities.

STO is VIN-centric, but upstream systems may not always send a real VIN. They may send:

- VIN
- PVIN
- pseudo vehicle ID
- regional vehicle reference
- source-system-specific identifier

The Dispatcher must resolve this into a canonical vehicle context.

A vehicle context may include:

- VIN
- brand
- country
- region
- home backend
- market
- ICTO/backend routing info

In Java/Spring terms, this would be a service like:

```text
VehicleIdentityResolutionService
    resolve(rawVehicleIdentifier, brand, region, sourceSystem)
```

Possible resolution strategies:

```text
1. Identity strategy
   If incoming identifier is already VIN, use it directly.

2. VDS home-region lookup
   Call external VDS to determine VIN and regional context.

3. VWAC lookup
   Use VWAC/vehicle-management-style service for certain regions or brands.

4. Composite strategy
   First resolve PVIN to VIN, then call another service for home-region context.
```

The selection of strategy is configuration-driven.

Why this matters:

> If the wrong VIN is resolved, the wrong customer could see the wrong event. Vehicle identity resolution is therefore a correctness-critical part of the system.

---

## 3.8 PVIN Cache

External vehicle resolution calls are expensive and potentially high-latency. Dispatcher therefore uses a local cache.

Spring Boot equivalent:

- Caffeine cache
- TTL expiration
- max size
- LRU-style eviction
- per-pod local cache

Flow:

```text
Need VIN for PVIN
        |
        v
Check local cache
        |
        +-- hit -> return VIN/context
        |
        +-- miss -> call VDS/VWAC
                    |
                    v
              cache result
                    |
                    v
              continue processing
```

Why local cache instead of distributed cache?

- Lower latency
- No extra network hop
- Simpler operationally
- Cache miss is acceptable
- Resolution data is usually stable enough for TTL caching

Interview phrasing:

> We intentionally kept the PVIN cache local to the pod. It avoided introducing another distributed dependency in the hot ingestion path. A new pod starts with a cold cache, but warms up naturally as events arrive.

---

## 3.9 Routing Configuration

After normalization and vehicle resolution, Dispatcher applies event configuration.

Configuration can determine:

- whether an event is relevant
- whether it should be sent to Supplier
- whether it should be visible to frontend
- whether it should be visible in vehicle
- whether it can trigger notification
- whether it is resettable
- which reset destination applies
- how to classify criticality
- how to map source event patterns

This configuration is typically consumed from a config topic or loaded from config resources and cached in memory.

Architecturally:

```text
Config source
        |
        v
Config consumer / loader
        |
        v
In-memory config cache
        |
        v
Dispatcher/Supplier business rules
```

This is crucial because automotive event behavior changes by:

- market
- region
- brand
- model generation
- event type
- legal policy
- customer channel
- campaign type

Hardcoding this would be too rigid.

---

## 3.10 Publishing Internal Events

Once Dispatcher has built the canonical internal event, it publishes it to an internal Kafka topic.

Important design choice:

> The Kafka message key should be VIN or a vehicle-based key.

Why?

Kafka preserves ordering within a partition. If all events for the same VIN use the same key, they go to the same partition, which helps preserve order for that vehicle.

```text
key = VIN
value = normalized service event
```

This helps reduce race conditions for the same vehicle.

---

## 3.11 Dispatcher Error Handling

Dispatcher classifies failures into categories.

### Bad Payload / Invalid Data

Examples:

- cannot deserialize
- missing required fields
- unknown event structure
- unsupported source message

Handling:

```text
Publish to DLQ/error topic
Commit original offset
Continue processing
```

Reason:

> Retrying the same malformed message forever would block good messages behind it.

---

### External Resolution Failure

Example:

- VDS unavailable
- VWAC timeout
- PVIN cannot be resolved

Handling may be:

```text
Retry with backoff
If still failing, publish to error topic
Commit original offset
Allow replay/error routine later
```

Reason:

> The event is not bad, but it cannot be processed right now. Parking it preserves it without blocking the stream.

---

### Internal Kafka Produce Failure

Example:

- Dispatcher cannot publish normalized event to internal topic

Handling:

```text
Do not commit original offset
Retry later / let Kafka redeliver
```

Reason:

> If the event was consumed but not published, committing would cause data loss.

---

## 3.12 Notification Flow in Dispatcher

Dispatcher also handles notification dispatch.

The notification flow is not the same as the raw event ingestion flow.

The general sequence:

```text
Supplier detects meaningful active customer-relevant event
        |
        v
Supplier publishes notification trigger
        |
        v
Dispatcher consumes notification trigger
        |
        v
Dispatcher checks notification configuration
        |
        v
Dispatcher calls FNS/MNP/notification endpoint
        |
        v
Customer receives push notification
```

Why Dispatcher handles notification instead of Supplier:

- Dispatcher already owns outbound routing concerns.
- Supplier should focus on state and APIs.
- Notification policy may be source/channel-specific.
- Avoid tightly coupling Cassandra write path with push delivery implementation.

Important:

> Supplier decides “something important changed.” Dispatcher decides “how and where to notify.”

---

## 3.13 Reset Flow in Dispatcher

Reset is a command path.

A customer or vehicle requests a reset through Supplier. Supplier validates the request and publishes a reset command. Dispatcher then routes that command to the correct upstream source system.

Flow:

```text
Client calls Supplier reset API
        |
        v
Supplier validates and publishes reset request
        |
        v
Dispatcher consumes reset request
        |
        v
Dispatcher maps event/source to external reset destination
        |
        v
Dispatcher publishes reset trigger to upstream system
        |
        v
Upstream system later emits updated event
```

This is architecturally clean because:

- Supplier owns public API.
- Dispatcher owns upstream topology.
- The frontend does not need to know source-system routing.
- Reset processing is asynchronous and resilient.

---

# 4. Supplier Service — Technical Deep Dive

## 4.1 Role of Supplier

Supplier is the **state owner and API layer**.

It consumes clean internal events and stores the current VIN-specific state in Cassandra. It also exposes REST APIs for:

- frontend/myAudi
- vehicle/in-vehicle systems
- workshop/service portlet
- external partners
- reset
- configuration
- DSGVO/GDPR
- health/readiness

In Spring Boot terms, Supplier contains:

- Kafka consumers
- service layer
- Cassandra repositories
- REST controllers
- security filters
- external REST clients
- response mappers
- refiner/enrichment pipeline
- health indicators

---

## 4.2 Supplier Has Two Modes

Supplier works in two main modes:

```text
1. Writer mode
   Kafka event -> Cassandra state

2. Reader mode
   REST request -> Cassandra + enrichment -> response
```

These two modes are related but different.

---

# 5. Supplier Writer Mode

## 5.1 Internal Event Consumption

Supplier consumes the normalized internal service event topic produced by Dispatcher.

Pipeline:

```text
Kafka normalized event
        |
        v
Deserialize internal event
        |
        v
Map to canonical service event model
        |
        v
Derive vehicle request context
        |
        v
Query existing Cassandra state
        |
        v
Apply idempotency decision
        |
        v
Write/update/delete/ignore
        |
        v
Optionally trigger notification
        |
        v
Commit Kafka offset
```

Spring Boot equivalent:

```text
@ServiceEventConsumer

@KafkaListener(topics = "service_event")
public void consume(ConsumerRecord<String, ServiceEvent> record,
                    Acknowledgment ack) {
    processor.process(record.value());
    ack.acknowledge();
}
```

But the key is: acknowledgment happens only after business processing completes.

---

## 5.2 Canonical Event Model

Supplier maps incoming internal events to a canonical domain model.

That model represents things like:

- VIN
- brand
- region
- country
- model ID / event ID
- event status
- event timestamp
- criticality
- display flags
- text references
- image references
- odometer
- remaining lifetime
- reset metadata
- recall metadata
- source information

The model is designed around what Supplier needs to store and serve, not necessarily around what the upstream source originally sent.

---

## 5.3 Cassandra as Current-State Store

Supplier uses Cassandra as the current state database.

The key concept:

> Cassandra stores the latest known state of an event for a vehicle, not every historical event.

The primary query pattern is:

```text
Give me all relevant events for VIN X.
```

Therefore the model is VIN-centric.

Conceptually:

```text
Vehicle VIN = WAU...
    Event W042 -> active
    Event R001 -> recall open
    Event Q123 -> quality info active
```

This is why Cassandra fits:

- high write throughput
- fast partition-key reads
- high availability
- TTL support
- horizontal scalability

---

## 5.4 Idempotency and Ordering Protection

This is one of the most important technical designs in Supplier.

Kafka is typically **at-least-once**. That means Supplier must handle:

- duplicate messages
- redelivery after failure
- delayed events
- out-of-order events
- upstream retries
- same event arriving multiple times

If Supplier blindly writes every message, bad things happen.

Example:

```text
Current Cassandra state:
VIN=ABC
event=R001
status=NOK
timestamp=14:00

Old Kafka event arrives:
status=OK
timestamp=13:50

If blindly written:
Customer incorrectly sees recall resolved.
```

So Supplier uses idempotency logic.

---

## 5.5 Idempotency Decision Logic

The logic is conceptually:

```text
Incoming event for VIN + event/model ID
        |
        v
Read existing Cassandra record
        |
        v
Decision:
```

| Existing state     | Incoming event                                   | Action                   |
| ------------------ | ------------------------------------------------ | ------------------------ |
| No existing record | Any valid event                                  | Insert                   |
| Existing record    | Incoming timestamp newer                         | Update                   |
| Existing record    | Incoming timestamp older                         | Ignore                   |
| Existing record    | Same/close timestamp, no meaningful change       | Ignore                   |
| Existing record    | Same/close timestamp, status/criticality changed | Update                   |
| Existing record    | Incoming status passive                          | Mark passive / short TTL |

This creates **business-level exactly-once semantics**.

Important nuance:

> Kafka may deliver the same message more than once, but the final Cassandra state remains correct.

---

## 5.6 Offset Commit Strategy

This is a critical reliability concept.

Supplier should commit Kafka offsets only after:

- event parsed successfully
- idempotency decision completed
- Cassandra write succeeded if needed
- any required follow-up publish completed or safely handled

If Cassandra is down:

```text
Do not commit offset
Kafka redelivers later
Idempotency handles repeat safely
```

This avoids data loss without needing distributed transactions.

Interview explanation:

> We avoided two-phase commit by making the Cassandra write idempotent and treating the Kafka offset as the processing boundary. If persistence failed, the offset was not committed, and Kafka replayed the event later.

---

## 5.7 TTL and Lifecycle Management

Supplier applies lifecycle rules at write time.

Event states have different lifetimes.

Conceptually:

| Status         | Lifecycle                      |
| -------------- | ------------------------------ |
| Active/NOK     | Keep longer                    |
| OK/resolved    | Keep for some time             |
| Passive/hidden | Keep briefly or expire quickly |
| GDPR deletion  | Explicit delete                |

Why TTL is important:

- avoids accumulating stale state
- reduces need for cleanup jobs
- keeps API responses relevant
- allows lifecycle to be config-driven

Cassandra TTL is useful because:

```text
Supplier writes row with TTL
Cassandra automatically expires it
```

Normal expiration is automatic. Legal deletion is explicit.

---

## 5.8 Notification Trigger from Supplier

Supplier may publish a notification trigger after a meaningful update.

Important condition:

> Notifications should only be triggered when the state actually changes in a meaningful way.

If a duplicate Kafka message arrives, Supplier should not send another push notification.

So notification trigger should happen after:

```text
idempotency decision = insert/update
AND status = active/NOK
AND event config says notification eligible
AND channel/criticality rules allow it
```

This avoids duplicate customer notifications.

---

# 6. Supplier Reader Mode — REST API Layer

Supplier exposes consumer-facing APIs.

Technically, in Spring Boot this maps to:

- `@RestController`
- request DTOs
- response DTOs
- service layer
- security filters
- WebClient clients
- Cassandra repositories
- response mappers

Consumer categories:

```text
1. Frontend/myAudi
2. Vehicle / in-vehicle app
3. Workshop / service portlet
4. External partner APIs
5. Reset APIs
6. Configuration APIs
7. DSGVO/GDPR APIs
8. Health/readiness APIs
```

---

## 6.1 Request Processing Flow

Typical REST request flow:

```text
HTTP request
        |
        v
Authentication / authorization
        |
        v
Build request context
        |
        v
Query Cassandra
        |
        v
Apply channel filters
        |
        v
Optional external enrichment
        |
        v
Localization
        |
        v
Response mapping
        |
        v
HTTP response
```

---

## 6.2 Vehicle Request Context

Supplier builds a request context containing:

- VIN
- brand
- country
- language
- region
- channel
- requested API version
- requested include parameters
- authentication/user context

This context influences:

- what records are read
- what data is visible
- what language is used
- what external systems are called
- which response version is returned
- what security constraints apply

---

## 6.3 Channel-Specific Filtering

Supplier does not return the same data to every consumer.

Examples:

### Frontend/myAudi

Return only customer-visible events.

```text
showInFrontend = true
```

### Vehicle

Return only vehicle-displayable events.

```text
showInVehicle = true
```

### Workshop

May return a broader or differently shaped view.

### Admin/config/compliance

Returns operational or governance data.

This is important because the same backend state may be transformed into different channel experiences.

---

## 6.4 Response Building

Supplier response building involves:

- filtering
- sorting
- deduplication
- localization
- status mapping
- version-specific response mapping
- hiding internal fields
- shaping data for consumer needs

In a Spring Boot design, this could be implemented as:

```text
Controller
    -> Application Service
        -> Query Services
        -> Refiner Chain
        -> Response Mapper
```

---

# 7. Refiner / Enrichment Pipeline

One of the most advanced parts of Supplier is the enrichment flow.

Supplier does not just return Cassandra rows. It may enrich responses from multiple sources:

```text
Cassandra STO events
Cassandra QCI events
Recall/OCC external API
ORU update status
CMS/zFDI localized text
Configuration and priority mapping
```

This is effectively a **chain-of-responsibility** or **pipeline**.

---

## 7.1 Conceptual Refiner Chain

```text
Base request context
        |
        v
STO Event Refiner
        |
        v
QCI Refiner
        |
        v
Recall/OCC Refiner
        |
        v
ORU Refiner
        |
        v
Localization Refiner
        |
        v
Filtering / Priority / Deduplication
        |
        v
Response DTO
```

Each refiner contributes data or modifies the response.

---

## 7.2 STO Event Refiner

This part reads the current STO service events from Cassandra.

It answers:

```text
What STO-managed warnings/service events exist for this VIN?
```

It applies:

- status filtering
- channel filtering
- brand/region rules
- visibility configuration

---

## 7.3 QCI Refiner

QCI means Quality Customer Information.

This refiner includes quality/customer information events relevant to the VIN.

It is separate because QCI events may have:

- different upstream source
- different lifecycle
- different display rules
- different business meaning

---

## 7.4 Recall / OCC Refiner

This is highly relevant to your resume point about SOAP-to-REST migration.

The recall/OCC refiner enriches response data with recall campaign information.

The old approach was likely:

```text
Supplier -> SOAP middleware/service -> Recall backend
```

The improved approach:

```text
Supplier -> direct REST API over mutual TLS -> Recall backend
```

This cuts latency because:

- removes SOAP envelope overhead
- removes middleware translation
- reduces serialization/deserialization layers
- avoids extra network hop
- uses direct HTTP/JSON-style interaction
- allows more targeted API calls

---

## 7.5 Recall REST Integration with mTLS

The direct recall integration likely works conceptually like this:

```text
Supplier OCC request
        |
        v
Need recall campaigns for VIN
        |
        v
Get OAuth/Cloud IDP token if required
        |
        v
Use WebClient/RestTemplate configured with client certificate
        |
        v
Call recall REST endpoint over mutual TLS
        |
        v
Receive campaign response
        |
        v
Map campaign response into STO/OCC response model
```

mTLS means both sides authenticate:

- Supplier validates server certificate.
- Recall service validates Supplier client certificate.

In Spring Boot terms:

```text
WebClient
    SSLContext with keyStore + trustStore
    client cert for mutual TLS
    token header if required
```

Benefits:

- stronger service-to-service authentication
- direct secure integration
- lower latency
- fewer moving parts

---

## 7.6 Feature Flag Rollback

Your resume mentions:

> zero-restart rollback via feature flag

This is a strong architectural point.

The migration from SOAP to REST likely used a toggle:

```text
if featureFlag.useDirectRecallRestApi:
    call REST recall API with mTLS
else:
    call legacy SOAP integration
```

In Spring Boot architecture:

```text
RecallClient
    LegacySoapRecallClient
    DirectRestRecallClient

RecallFacade
    if featureFlag enabled:
        return directRestRecallClient.getCampaigns(vin)
    else:
        return legacySoapRecallClient.getCampaigns(vin)
```

Why this matters:

- No redeployment required.
- No restart required.
- Fast rollback if REST dependency misbehaves.
- Safe gradual rollout by environment/market/percentage.
- Production risk reduced.

Interview phrasing:

> We wrapped the new REST recall integration behind a feature flag so we could switch back to the legacy SOAP path instantly without restarting the service. That allowed us to migrate safely while reducing latency by removing the SOAP translation layer and calling the recall API directly over mTLS.

---

## 7.7 Partial Response Strategy

External enrichment can fail. Recall service may be down or slow.

Supplier should not always fail the entire API response.

Instead:

```text
If STO data is available
AND recall enrichment fails
THEN return partial response
```

Possible HTTP behavior:

- `200 OK` when all sources succeed
- `206 Partial Content` when some enrichment is missing
- `5xx` only when core response cannot be served

This protects user experience.

Business reason:

> Showing locally available warnings is better than showing a blank screen because one external recall system is down.

---

## 7.8 ORU Refiner

ORU/update status enrichment adds online update or campaign status information.

It may map external states to customer-facing states such as:

- preparation
- ready
- workshop
- done
- unavailable

Flow:

```text
Recall/OCC event identified as ORU-relevant
        |
        v
Call ORU/update status service
        |
        v
Map technical status to STO response status
        |
        v
Merge into response
```

---

## 7.9 CMS / zFDI Localization

Events usually should not carry full text in all languages. Instead, they carry references:

- text ID
- picture ID
- detail text ID
- language/country context

Supplier resolves these references into localized customer-facing content.

Flow:

```text
Event contains txtId
        |
        v
Request language = de_DE
        |
        v
Lookup CMS/zFDI content
        |
        v
Return localized title/detail/action text
```

Caching is important because localized CMS content is relatively static compared to request frequency.

Spring Boot equivalent:

```text
CmsDataService
    -> cache lookup
    -> fallback/local resource lookup
    -> external zFDI/CMS refresh if stale
```

---

# 8. Supplier API Categories

## 8.1 Frontend APIs

Used by:

- myAudi app
- web portal

Responsibilities:

- return VIN-specific active events
- apply frontend visibility
- localize text
- include recall/quality information if requested
- shape response for customer UI

Security:

- OAuth/JWT
- customer/application scope
- VIN authorization context

---

## 8.2 Vehicle APIs

Used by:

- in-vehicle systems
- infotainment/head-unit applications
- ignition-time sync flows

Responsibilities:

- return events suitable for vehicle display
- apply vehicle visibility
- possibly use stricter vehicle context validation
- return compact/vehicle-specific format

---

## 8.3 External / Service Portlet APIs

Used by:

- workshops
- partners
- service advisors

Responsibilities:

- return VIN service/campaign state
- support workshop planning
- may expose partner-specific response structure

Security:

- partner/system credentials
- OAuth or basic auth depending on route
- possibly host or network restrictions

---

## 8.4 Reset APIs

Used by frontend/vehicle to request reset.

Supplier responsibilities:

- authenticate user
- validate event exists
- check reset eligibility
- publish reset command
- return async acceptance

Response should usually be asynchronous:

```text
202 Accepted
```

because actual reset depends on upstream source system.

---

## 8.5 Configuration APIs

Used by internal tools.

Responsibilities:

- initialize configs
- retrieve configs
- publish config changes
- validate event config

Configuration affects:

- visibility
- TTL
- notification
- reset behavior
- event matching
- priority

---

## 8.6 DSGVO/GDPR APIs

Used by compliance/internal systems.

Responsibilities:

- inquiry: what data is stored
- delete by VIN
- delete by event type
- delete specific event
- support legal retention workflows

This is important because VIN/event data can be privacy-sensitive.

---

# 9. Mock Server / mockhttp4s — Technical Deep Dive

## 9.1 Purpose of Mock Server

The mock server simulates external enterprise dependencies.

It exists to make local development and CI deterministic.

Without the mock server, integration tests would depend on:

- real VDS service
- real VIN resolver
- real recall service
- real token provider
- real FNS notification service
- real ORU service

Those systems may be:

- unavailable
- slow
- rate-limited
- inconsistent
- require real credentials
- return changing data
- unsuitable for CI

The mock server solves this by providing controlled fake endpoints.

---

## 9.2 Dependencies Simulated

The mock server simulates dependencies such as:

```text
1. VDS / vehicle data service
2. VIN/PVIN resolver
3. Recall campaign service
4. Cloud IDP / token endpoint
5. FNS notification endpoint
6. ORU update status service
```

Your resume says “4 external enterprise dependencies.” You can safely frame it as:

> I built the mock server to simulate the key external enterprise dependencies used in the end-to-end flow: vehicle identity resolution, recall campaign lookup, authentication/token provider, and notification/update-status services.

If asked why the repository shows more mocks, say:

> The mock server evolved to simulate more dependencies, but the core CI stabilization work focused on the four dependencies that caused most pipeline flakiness.

---

## 9.3 Mock Server Architecture

The mock server behaves like a standalone Spring Boot service in your explanation.

It has:

- REST controllers for fake external endpoints
- configurable response behavior
- static/dynamic test data
- Kafka producer endpoints to inject events
- failure simulation support
- latency/error configuration
- request counting/verification for assertions

Conceptual layout:

```text
Mock Server
    |
    +-- VDS mock endpoint
    +-- VIN resolver mock endpoint
    +-- Recall campaigns mock endpoint
    +-- Token mock endpoint
    +-- FNS mock endpoint
    +-- ORU mock endpoint
    +-- Kafka event injection endpoint
```

---

## 9.4 Event Injection

The mock server does more than return fake HTTP responses.

It can also inject Kafka events into the STO pipeline.

Flow:

```text
Integration test calls mock endpoint:
PUT /ACDCEventAsJSON
        |
        v
Mock server publishes Kafka message
        |
        v
Dispatcher consumes message
        |
        v
Supplier stores state
        |
        v
Test calls Supplier API
        |
        v
Assert response
```

This enables true end-to-end tests.

---

## 9.5 Failure Simulation

A major benefit is controlled failure testing.

Mock server can simulate:

- HTTP 500
- HTTP 503
- timeout
- slow response
- invalid payload
- empty response
- unauthorized token response
- missing vehicle data
- recall service unavailable

This allows testing:

- retry behavior
- DLQ routing
- partial responses
- fallback behavior
- feature flag rollback
- health check behavior
- notification failure handling

This is the basis for your resume point:

> Built a WireMock-based mock server simulating external enterprise dependencies, reducing CI pipeline failures caused by external service instability.

---

## 9.6 Why CI Failure Rate Improved

Before mock server:

```text
CI test -> real external dependency
             |
             +-- dependency down -> CI fails
             +-- slow response -> timeout
             +-- changed data -> assertion fails
             +-- auth issue -> test fails
```

After mock server:

```text
CI test -> mock dependency
             |
             +-- deterministic response
             +-- controlled latency
             +-- controlled errors
             +-- stable data
```

This reduces flaky tests dramatically.

Interview phrasing:

> The problem was not that our application logic was failing; CI was failing because external enterprise systems were unavailable or inconsistent. By introducing a mock server with deterministic responses and Kafka event injection, we made the pipeline self-contained and reduced external-dependency-related CI failures by around 80%.

---

# 10. Common Libraries and Supporting Packages

Now let’s understand the supporting modules architecturally.

---

# 10.1 02 STO Commons Library

This is the shared foundation library.

It contains reusable components for:

- Kafka
- HTTP clients
- serialization/deserialization
- CMS data
- config
- health checks
- protobuf/avro models
- tracing/logging
- activation/blue-green behavior
- utility models like brand/region/context

Architecturally, this avoids duplicating common logic across Dispatcher, Supplier, routines, and tests.

---

## 10.1.1 Shared Kafka Components

The STO Kafka module provides:

- Kafka consumer abstractions
- Kafka producer abstractions
- serializers/deserializers
- error-handling deserializers
- pipeline abstractions
- offset management helpers
- liveness watchdog
- Kafka admin utilities
- producer config helpers

Spring Boot equivalent:

- common `KafkaConsumerConfig`
- common `KafkaProducerConfig`
- custom deserializers
- reusable error handling
- producer wrapper service
- consumer health/liveness monitor

Why this matters:

> Both Supplier and Dispatcher use Kafka heavily. Shared Kafka utilities enforce consistent behavior around serialization, offset handling, logging, liveness, and configuration.

---

## 10.1.2 Shared HTTP Client Components

The HTTP client module provides reusable clients for:

- FNS notification
- recall service
- ORU service
- Cloud IDP
- zFDI
- token services
- SSL/mTLS config
- OAuth token handling
- proxy config

Spring Boot equivalent:

- shared `WebClient` builders
- mTLS-enabled HTTP client factory
- OAuth token provider
- retry/rate-limit wrappers
- health-checking access token provider

Why this matters:

> External enterprise calls need consistent authentication, TLS, proxy, retry, and health behavior across services.

---

## 10.1.3 Config Module

The config module models STO event configuration.

It includes concepts such as:

- notification config
- event TTL
- push config
- vehicle meta config
- config cache
- pattern matching
- fallback config
- config producer/consumer

This is central to business behavior.

Conceptual flow:

```text
Config message arrives
        |
        v
Config processor validates/parses
        |
        v
Config cache updates
        |
        v
Dispatcher/Supplier use new rules
```

This enables dynamic behavior without redeployment.

---

## 10.1.4 CMS Data Provider

The utility module includes CMS data provider functionality.

It supports:

- language metadata
- warning text data
- localized messages
- fallback behavior
- file-system/resource-based CMS loading
- parsing and validation of CMS data

Supplier uses this concept to return localized content.

---

## 10.1.5 Telemetry and Tracing

The commons library includes logging/tracing helpers.

Concepts:

- W3C trace context
- MDC enrichment
- Kafka log correlation
- HTTP middleware
- OpenTelemetry-style propagation

This is important because an event crosses:

```text
Kafka -> Dispatcher -> Kafka -> Supplier -> Cassandra -> external HTTP -> API response
```

Without trace propagation, diagnosing production issues would be painful.

---

## 10.1.6 Protobuf and Avro Libraries

STO uses shared schema/model libraries for:

- vehicle notification protocol
- gateway event models
- service notification models
- recall notification models
- reset notification models
- FNS notification models

These provide schema consistency across producers/consumers.

In Spring Boot language:

> These libraries are shared contract artifacts used by services to serialize and deserialize event payloads safely.

---

## 10.1.7 Plumber / Activation

The plumber module supports active/standby or blue/green runtime activation.

Conceptually:

```text
Pod color = blue/green
Active color = blue
        |
        +-- blue pods process traffic
        +-- green pods pause consumers / return unavailable for business routes
```

This prevents dual-active consumption.

This matters because Kafka consumers and Cassandra writes are stateful. If both colors process the same topics incorrectly, duplicate or conflicting processing can happen.

---

# 10.2 03 Cassandra Commons

Cassandra commons provides:

- Cassandra connection handling
- SSL/auth options
- regional connectors
- Cassandra data models
- schema migration tooling
- TTL validation
- Avro-to-Cassandra model mapping
- deletion result models
- region loading

Architecturally, this module owns the database foundation.

---

## 10.2.1 Cassandra Connectors

These components abstract Cassandra session creation and connection configuration.

They handle:

- contact points
- keyspace/region selection
- authentication
- SSL/mTLS-like Cassandra connection security
- regional cluster connection

Spring Boot equivalent:

- Cassandra configuration beans
- `CqlSession`
- `CassandraTemplate`
- environment-specific Cassandra properties

---

## 10.2.2 Cassandra Data Models

The models represent persisted domain state:

- generic event
- service event versions
- quality customer information event
- OCC event with criteria
- region/model version/deletion results

These are the persistence model for Supplier.

Important:

> The data model evolved over time, which is why there are versioned service event models.

This reflects real enterprise evolution:

- new event fields
- recall columns
- reset columns
- detail texts
- new AVRO/FNS formats
- region-specific tables

---

## 10.2.3 Cassandra Migration Tooling

The migration packages manage schema evolution.

In production systems, Cassandra schemas must evolve carefully.

Migration tooling supports:

- versioned schema migrations
- schema version table
- migration ordering
- schema agreement checks
- data migrations
- regional schema setup
- adding columns/tables over time

Why important:

> STO evolved through multiple product levels and event formats. Cassandra schema migration tooling allowed controlled upgrades without manually applying inconsistent database changes.

---

## 10.2.4 TTL Validator

TTL validation ensures configured TTLs are valid and safe.

This matters because bad TTL config could cause:

- customer-visible events disappearing too early
- old events staying too long
- compliance/data-retention issues
- inconsistent event lifecycle

---

# 10.3 04 OAuth Validator Library

This module provides authentication and token validation.

It supports:

- JWT validation
- token filters
- token payload parsing
- token clause validation
- issuer/audience/expiry checks
- public key validation
- possibly Google/Auth token filter variants

Spring Boot equivalent:

- Spring Security filter
- JWT decoder
- authentication provider
- request authentication middleware

Supplier uses authentication heavily because APIs are exposed to different consumers.

Route categories require different access rules:

```text
Frontend route       -> customer/app token
Vehicle route        -> vehicle-scoped token
External route       -> partner/system token
Reset route          -> reset permission
Config route         -> admin/internal scope
DSGVO route          -> compliance/deletion scope
```

Technical validation includes:

- signature
- expiry
- issuer
- audience
- scopes/roles
- token claims
- possibly VIN/brand context

---

# 10.4 09 Commons

This module provides shared domain services and utilities used by STO services.

Key responsibilities:

- retry helper
- PVIN cache manager
- VDS HTTP client
- VWAC client
- zFDI Cassandra service
- token service
- LogAcc service
- Kafka producer wrappers
- internal producer message models
- vehicle context models
- trace headers
- HTTP config loaders

This is a practical shared service layer.

---

## 10.4.1 Retry Utility

External calls need retry logic.

Common retry behavior:

```text
try call
if transient failure:
    retry with delay/backoff
if permanent failure:
    fail/route error
```

Used for:

- VDS
- VWAC
- token service
- notification calls
- recall calls

---

## 10.4.2 PVIN Cache Manager

This likely provides shared cache logic for PVIN/VIN resolution.

Used mostly by Dispatcher.

Concept:

```text
PvinMapKey -> VehicleContext
```

The cache protects external vehicle identity services.

---

## 10.4.3 VDS / VWAC Clients

These are external clients for vehicle context resolution.

Dispatcher depends on these to map upstream identifiers to usable vehicle context.

In Spring Boot:

- `VdsClient`
- `VwacClient`
- WebClient/RestTemplate
- OAuth/basic/mTLS config as needed
- retry and timeout settings

---

## 10.4.4 zFDI Service

zFDI appears to be tied to CMS/localized content.

Supplier uses it to enrich events with customer-readable detail text.

---

## 10.4.5 Shared Kafka Producer

Common producer wrappers help enforce consistent Kafka publishing behavior:

- topic config
- serialization
- logging
- error handling
- internal producer messages

---

# 10.5 01 Datahub Commons

This appears to provide certificate utility support.

In the architecture, it supports secure integration, especially:

- loading certificates
- parsing certs
- trust material handling
- mTLS-related utilities

This is relevant to your recall migration point.

For mTLS REST calls, the service needs:

- client certificate
- private key or keystore
- truststore
- SSL context
- certificate validation

A certificate utility module helps standardize this.

---

# 10.6 11 STO Integration Tests

The integration test module validates system behavior end-to-end.

It likely uses:

- Docker Compose / Testcontainers
- Kafka
- Cassandra
- Dispatcher
- Supplier
- Mock server
- SSL Cassandra variants
- test payloads
- full roundtrip tests

Test categories include:

```text
ACDC event roundtrip
Active/standby switch
DSGVO roundtrip
Event configuration roundtrip
External interface test
Frontend notification test
Healthcheck page test
```

These tests prove that the architecture works as a system, not just as isolated units.

---

## 10.6.1 ACDC Roundtrip Test

Validates:

```text
Mock/injected ACDC event
        |
        v
Dispatcher processing
        |
        v
Supplier Cassandra write
        |
        v
Supplier API response
```

This is the most important system test pattern.

---

## 10.6.2 Active/Standby Switch Test

Validates blue/green behavior:

- active pod processes
- standby does not process
- switching activation works
- health endpoints behave correctly
- consumer behavior remains safe

---

## 10.6.3 DSGVO Roundtrip Test

Validates:

- data inquiry
- deletion request
- Cassandra state removal
- API confirms deletion

---

## 10.6.4 Event Configuration Roundtrip

Validates:

- config update is accepted
- config is distributed/loaded
- event behavior changes accordingly

This proves config-driven architecture.

---

## 10.6.5 Frontend Notification Test

Validates:

```text
NOK event stored
        |
        v
notification trigger produced
        |
        v
Dispatcher calls notification mock
        |
        v
test verifies call/count
```

This proves notification side-effect flow.

---

# 11. End-to-End Technical Flows

Now let’s walk through key flows technically.

---

# 11.1 Flow 1 — New Active Event to Customer App

```text
1. Upstream system publishes raw event to Kafka.

2. Dispatcher consumes message.

3. Dispatcher deserializes payload.

4. Dispatcher validates required fields.

5. Dispatcher normalizes event.

6. Dispatcher resolves PVIN/VIN.
   - cache lookup
   - external VDS/VWAC call if needed

7. Dispatcher loads relevant config.
   - visibility
   - notification
   - resetability
   - event behavior

8. Dispatcher publishes normalized service event to internal topic.

9. Supplier consumes normalized event.

10. Supplier maps to canonical service event.

11. Supplier reads existing Cassandra state.

12. Supplier applies idempotency.
    - insert/update/ignore

13. Supplier writes Cassandra with TTL.

14. Supplier emits notification trigger if needed.

15. Dispatcher consumes notification trigger.

16. Dispatcher calls notification endpoint.

17. Customer opens app.

18. Supplier API authenticates request.

19. Supplier reads Cassandra by VIN.

20. Supplier enriches with localized text.

21. Supplier filters for frontend.

22. Supplier returns response.
```

---

# 11.2 Flow 2 — Recall Enrichment with REST+mTLS

This flow connects directly to your resume.

```text
1. Client calls Supplier OCC/frontend API for VIN.

2. Supplier authenticates request.

3. Supplier builds request context:
   VIN, brand, market, language, channel.

4. Supplier reads locally stored STO events from Cassandra.

5. Supplier determines recall data is needed.

6. Supplier checks feature flag:
   - if direct REST enabled -> use new REST recall client
   - if disabled -> use legacy SOAP path

7. Direct REST path:
   - get/cached token if required
   - build mTLS-enabled HTTP client
   - call recall REST endpoint
   - receive campaign response

8. Supplier maps recall campaigns to OCC response model.

9. Supplier optionally stores recall data in Cassandra cache.

10. Supplier enriches recall with ORU/CMS data if needed.

11. Supplier combines STO + QCI + recall data.

12. Supplier returns response.
```

Latency reduction comes from:

- direct REST instead of SOAP
- less XML parsing
- fewer intermediate services
- fewer transformations
- fewer network hops
- persistent HTTP connection reuse
- cleaner payload contract

Rollback safety comes from feature flag:

```text
feature flag off -> legacy SOAP
feature flag on  -> direct REST
```

No restart needed if feature config is dynamic.

---

# 11.3 Flow 3 — Reset Command

```text
1. Customer taps reset.

2. Frontend calls Supplier reset endpoint.

3. Supplier authenticates user.

4. Supplier validates:
   - event exists
   - event belongs to VIN
   - reset allowed by config
   - channel is allowed to reset

5. Supplier publishes reset request to Kafka.

6. Supplier returns 202 Accepted.

7. Dispatcher consumes reset request.

8. Dispatcher determines source system for event.

9. Dispatcher resolves PVIN if source needs PVIN instead of VIN.

10. Dispatcher publishes reset trigger to external source topic.

11. Source system processes reset.

12. Source emits updated OK/passive event.

13. Normal Dispatcher -> Supplier path updates Cassandra.
```

This is a clean async command/event loop.

---

# 11.4 Flow 4 — Bad Event Handling

```text
1. Dispatcher receives malformed event.

2. Deserialization or validation fails.

3. Dispatcher creates error record.

4. Dispatcher publishes to error/DLQ topic.

5. Dispatcher commits original offset.

6. Pipeline continues with next message.
```

Why commit?

> Because retrying a malformed message will not fix it and would block the partition.

---

# 11.5 Flow 5 — Cassandra Failure

```text
1. Supplier consumes valid event.

2. Supplier determines Cassandra write required.

3. Cassandra is unavailable.

4. Supplier does not commit Kafka offset.

5. Kafka redelivers later.

6. When Cassandra recovers, Supplier processes event.

7. Idempotency ensures duplicate processing is safe.
```

This is a core reliability pattern.

---

# 12. Security Architecture

## 12.1 Inbound Security

Supplier APIs require authentication.

In Spring Boot terms:

```text
HTTP request
        |
        v
Security filter chain
        |
        v
JWT validation
        |
        v
Scope/role validation
        |
        v
Request context extraction
        |
        v
Controller
```

Validation checks:

- signature
- issuer
- audience
- expiry
- issued-at
- scopes
- token version
- claims
- possibly VIN/brand/country authorization

Different routes have different scopes.

---

## 12.2 Outbound Security

Outbound calls may require:

- OAuth token
- mTLS
- basic auth
- proxy config
- custom headers
- trace headers

For recall REST migration, mTLS is key.

Technical setup:

```text
Keystore contains Supplier client cert/private key
Truststore contains trusted CA/server certs
HTTP client uses SSLContext
Recall service validates client certificate
Supplier validates recall server certificate
```

This is stronger than simple bearer-token-only authentication.

---

# 13. Observability

Observability spans:

- structured logs
- trace IDs
- Kafka headers
- HTTP headers
- health checks
- liveness watchdogs
- audit trail
- log access service

## 13.1 Trace Propagation

Trace context flows through:

```text
Inbound Kafka headers
        |
        v
Dispatcher processing logs
        |
        v
Internal Kafka headers
        |
        v
Supplier processing logs
        |
        v
Outbound HTTP headers
        |
        v
External systems
```

Logs include:

- VIN
- event ID
- brand
- country
- trace ID
- source system
- status
- topic/partition/offset

This enables:

> “Show me everything that happened for VIN X or trace ID Y.”

---

## 13.2 Health Checks

Health checks cover:

- Kafka consumer health
- Kafka producer health
- Cassandra connectivity
- external dependency readiness
- config availability
- CMS freshness
- token provider health

In Spring Boot:

- `/actuator/health/liveness`
- `/actuator/health/readiness`
- custom `HealthIndicator`s

---

# 14. Blue/Green / Active-Standby

The activation model prevents dual-active processing.

Concept:

```text
Two deployment colors:
blue
green

Only active color:
- consumes Kafka
- serves business APIs

Standby color:
- health endpoints available
- Kafka consumers paused
- business routes unavailable/503
```

Why this matters:

If both colors consume and write at the same time:

- duplicate processing
- duplicate notifications
- possible race conditions
- operational confusion

Consumer group IDs should be color-neutral so active switch continues from latest committed offset.

---

# 15. Your Resume Bullet 1 — Technical Explanation

Resume bullet:

> Developed REST APIs exposing VIN-specific vehicle notification data across multiple consumer channels and migrated a legacy SOAP-based recall integration to a direct REST API with mutual TLS authentication, cutting latency by ∼35% with zero-restart rollback via feature flag.

A strong technical explanation:

> I worked on the Supplier API layer, which exposes VIN-specific notification and service-event data to different channels like the myAudi frontend, vehicle systems, and service partners. The APIs read the current vehicle state from Cassandra, apply channel-specific visibility rules, enrich the response with campaign and localized text data, and return a consumer-specific response model.
>
> One major improvement was replacing a legacy SOAP-based recall integration with a direct REST integration. The previous path had SOAP/XML overhead and extra translation layers. We introduced a direct REST client secured with mutual TLS, so the Supplier could call the recall service directly using a client certificate and truststore-based validation.
>
> To reduce production risk, we placed the new REST client behind a feature flag. At runtime, the recall facade could choose either the new REST path or the old SOAP path. That allowed zero-restart rollback if the new integration had issues. The direct REST path reduced latency by around 35% by removing the SOAP middleware and XML transformation overhead.

Architecture diagram:

```text
Before:
Supplier API
    |
    v
Legacy SOAP client
    |
    v
SOAP middleware / translation
    |
    v
Recall backend

After:
Supplier API
    |
    v
Feature flag
    |
    +-- off -> legacy SOAP path
    |
    +-- on  -> direct REST client with mTLS
                    |
                    v
              Recall REST backend
```

---

# 16. Your Resume Bullet 2 — Technical Explanation

Resume bullet:

> Built a WireMock-based mock server simulating 4 external enterprise dependencies, reducing CI pipeline failure rate due to external service unavailability by ∼80%.

Strong technical explanation:

> Our integration tests depended on several external enterprise systems, such as vehicle identity resolution, recall campaign lookup, token generation, and notification/update-status services. These dependencies made CI flaky because tests failed when external services were unavailable, slow, or returned inconsistent data.
>
> I built a WireMock-style mock server that simulated these dependencies with deterministic responses. It also supported failure simulation, so we could test retry, fallback, DLQ, and partial-response behavior. In addition, the mock server could inject Kafka events through HTTP endpoints, which allowed us to trigger a complete Dispatcher-to-Supplier roundtrip in CI without real external systems.
>
> This made the CI pipeline self-contained and reduced failures caused by external dependency unavailability by around 80%.

Architecture:

```text
Before:
CI test
  -> Dispatcher/Supplier
      -> real VDS
      -> real recall API
      -> real IDP
      -> real FNS/ORU
  = flaky

After:
CI test
  -> Dispatcher/Supplier
      -> mock VDS
      -> mock recall API
      -> mock IDP
      -> mock FNS/ORU
  = deterministic
```

---

# 17. How to Present This as Java/Spring Boot

Even though the codebase is Scala, this maps naturally to Spring Boot.

## Dispatcher Spring Boot Mapping

```text
DispatcherApplication
    |
    +-- Kafka listeners
    +-- Event deserializer
    +-- Event validation service
    +-- Event normalization service
    +-- Vehicle identity resolution service
    +-- Routing config service
    +-- Internal event producer
    +-- Notification consumer/client
    +-- Reset consumer/producer
    +-- Health indicators
```

## Supplier Spring Boot Mapping

```text
SupplierApplication
    |
    +-- Kafka service event consumer
    +-- Idempotency service
    +-- Cassandra repositories
    +-- REST controllers
    +-- Security filter chain
    +-- Recall client
    +-- CMS/zFDI service
    +-- ORU client
    +-- Response refiner pipeline
    +-- Reset service
    +-- DSGVO service
    +-- Health indicators
```

## Mock Server Spring Boot Mapping

```text
MockServerApplication
    |
    +-- VDS mock controller
    +-- Recall mock controller
    +-- Token mock controller
    +-- FNS mock controller
    +-- ORU mock controller
    +-- Kafka event injection controller
    +-- Failure scenario configuration
```

---

# 18. Final Interview-Ready Technical Story

You can say:

> The system had two main services: Dispatcher and Supplier. Dispatcher was the ingestion service. It consumed raw Kafka events from upstream vehicle systems, parsed and validated them, resolved vehicle identity from PVIN to VIN using external vehicle services with caching, applied event configuration, and published normalized internal events.
>
> Supplier was the state and API service. It consumed those normalized events, applied idempotency logic against Cassandra, and stored the current VIN-specific event state. It also exposed REST APIs for frontend, vehicle, service partner, reset, configuration, and GDPR use cases. On read APIs, Supplier enriched data with recall campaign details, ORU status, and localized CMS/zFDI texts.
>
> The most important reliability pattern was combining manual Kafka offset handling with idempotent Cassandra writes. If Cassandra was down, offsets were not committed and Kafka redelivered later. If Kafka duplicated messages, Supplier compared timestamps and state before writing, so the final vehicle state remained correct.
>
> I also worked on modernizing the recall integration. We moved from a legacy SOAP-based call path to a direct REST API secured with mutual TLS. The new integration was hidden behind a feature flag so we could roll back instantly without restart. This reduced latency by about 35%.
>
> For CI stability, I built a WireMock-style mock server that simulated key external dependencies like vehicle resolution, recall, token provider, and notification/update-status services. It also supported Kafka event injection and failure simulation. That made integration tests deterministic and reduced CI failures due to external system unavailability by around 80%.

---

# 19. One-Line Technical Summary

> **Dispatcher normalizes and routes vehicle events; Supplier stores the latest VIN state and serves enriched APIs; Kafka provides decoupling and replay; Cassandra provides fast VIN-based state; feature flags, mTLS, mocks, and idempotency make the system production-grade.**
