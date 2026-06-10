Absolutely. Now that you narrowed the scope to the two main services, the system becomes much easier to own in an interview.

The two services are:

1. **`sto-dispatcher-service`** — ingestion, normalization, routing, reset forwarding, notification triggering.
2. **`sto-supplier-service`** — state persistence, idempotency, Cassandra ownership, REST API serving, enrichment, GDPR/config endpoints.

The simplest mental model:

> **Dispatcher receives and cleans vehicle events. Supplier stores and serves vehicle event state. Kafka connects them. Cassandra is the Supplier’s source of truth.**

---

# 1. Big Picture

```text architecture.txt
Upstream Systems
ACDC / Predictive / QCI / E3 / Recall-like feeds
        |
        | raw Kafka events
        v
+------------------------+
| sto-dispatcher-service |
|                        |
| - consume raw events   |
| - deserialize          |
| - validate             |
| - normalize            |
| - resolve VIN/PVIN     |
| - apply config         |
| - publish clean event  |
+------------------------+
        |
        | internal Kafka topic: service_event
        v
+----------------------+
| sto-supplier-service |
|                      |
| - consume clean event|
| - idempotency check  |
| - write Cassandra    |
| - publish notify msg |
| - expose REST APIs   |
| - enrich responses   |
+----------------------+
        |
        v
+-----------+
| Cassandra |
| VIN state |
+-----------+
        |
        v
Consumers:
myAudi app / portal / in-vehicle / workshops / DSGVO / admin
```

If you say only one thing in the interview, say this:

> The architecture separates **event ingestion** from **state ownership and API serving**. Dispatcher is stateless and optimized for high-throughput Kafka ingestion. Supplier is stateful and optimized for correctness, Cassandra persistence, and REST API responses.

---

# 2. Why Two Services?

This is one of the most important design decisions.

## Dispatcher and Supplier have different jobs

| Concern | Dispatcher | Supplier |
|---|---|---|
| Primary role | Ingest raw upstream events | Store and serve current vehicle state |
| Statefulness | Mostly stateless | Stateful |
| Owns Cassandra? | No | Yes |
| Owns REST APIs? | Minimal/health/admin style | Yes, main API layer |
| Main workload | Kafka in, Kafka out | Kafka in, Cassandra, REST out |
| Scaling bottleneck | CPU/network/external VIN lookup | Cassandra/API traffic/external enrichments |
| Failure impact | Ingestion/routing affected | Persistence/API affected |
| Main risk | Bad upstream data, VIN resolution failure | Duplicate/stale writes, dependency failure |

Interview version:

> We split the system because Dispatcher and Supplier have different scaling and reliability needs. Dispatcher handles bursty upstream Kafka traffic and normalizes events. Supplier handles durable state and customer-facing APIs. Kafka between them gives buffering, decoupling, independent deployment, and replayability.

---

# 3. `sto-dispatcher-service`

## 3.1 What Dispatcher Does

The Dispatcher is the **front door** of STO.

It consumes raw events from upstream Kafka topics and turns them into normalized internal events.

Its responsibilities are:

1. Consume raw upstream Kafka messages.
2. Deserialize payloads.
3. Validate required fields.
4. Normalize brand/region/event data.
5. Resolve vehicle identity, especially PVIN to VIN.
6. Apply routing/configuration rules.
7. Publish clean normalized events to the internal Kafka topic.
8. Route bad events to error/DLQ topics.
9. Forward reset requests to the correct upstream system.
10. Trigger customer notifications through FNS/MNP when required.

---

## 3.2 Dispatcher File Map

Based on the repository map, these are the important Dispatcher areas:

## Startup / shared service

- `src/main/scala/dispatcher/services/DispatcherStartService.scala`
- `src/main/scala/dispatcher/services/ApplicationSharedService.scala`
- `src/main/scala/dispatcher/services/HealthcheckService.scala`

These are the service bootstrapping and lifecycle pieces.

Spring Boot equivalent:

- `ApplicationRunner`
- `@Service`
- `HealthIndicator`
- actuator readiness/liveness checks

---

## Inbound Kafka processing

- `src/main/scala/dispatcher/services/inbound/acdc/InboundKafkaMsgProcessorFactory.scala`
- `src/main/scala/dispatcher/services/inbound/acdc/InboundMsgProcessorHandlerFactory.scala`
- `src/main/scala/dispatcher/services/inbound/acdc/IncomingMsgKafkaConsumerAdmin.scala`
- `kafka-util/src/main/scala/com/vw/sto/kafka/consumer/InboundKafkaMsgProcessor.scala`
- `kafka-util/src/main/scala/com/vw/sto/kafka/consumer/InboundMsgProcessorHandler.scala`
- `kafka-util/src/main/scala/com/vw/sto/kafka/consumer/impl/InboundMsgProcessorHandlerImpl.scala`
- `kafka-util/src/main/scala/com/vw/sto/kafka/consumer/parser/acdc/InboundKafkaMsgParser.scala`
- `kafka-util/src/main/scala/com/vw/sto/kafka/consumer/impl/StoKafkaConfig.scala`

This is the actual raw event consumer side.

Spring Boot equivalent:

- `@KafkaListener`
- deserializer
- event handler
- `KafkaTemplate`
- manual acknowledgment
- DLQ handling

Interview explanation:

> Dispatcher consumes ACDC and similar upstream events, parses the raw event envelope, validates it, enriches it with vehicle context, and publishes a canonical internal event.

---

## PVIN/VIN resolution

- `src/main/scala/dispatcher/services/PvinResolverStrategyProvider.scala`
- `src/main/scala/dispatcher/services/inbound/acdc/PvinCacheConfig.scala`
- `src/test/scala/dispatcher/services/PvinResolverStrategyProviderTest.scala`
- `src/test/scala/dispatcher/services/inbound/acdc/PvinCacheConfigTest.scala`
- `src/test/scala/dispatcher/services/inbound/acdc/PvinCacheManagerTest.scala`
- `src/main/scala/dispatcher/services/outbound/vds/VdsClientBuilder.scala`
- `src/main/scala/dispatcher/services/outbound/vwac/VwAcService.scala`

This is one of the most interesting technical areas.

Business reason:

> Upstream systems may send a pseudonymous vehicle identifier, PVIN, instead of the actual VIN. Dispatcher resolves that to the canonical VIN and region context before routing the event.

Spring Boot equivalent:

- `VinResolutionService`
- `VdsClient`
- `VwacClient`
- `CaffeineCache`
- `WebClient`
- retry/backoff logic

Interview phrase:

> A key part of Dispatcher was vehicle identity resolution. Some upstream systems did not send a real VIN; they sent a PVIN. We supported multiple resolution strategies and cached successful resolutions locally to avoid calling external services for every Kafka event.

---

## Config retrieval

- `src/main/scala/dispatcher/services/outbound/internal/config/ConfigRetriever.scala`
- `src/main/scala/dispatcher/services/outbound/internal/config/ConfigMessagePlayService.scala`
- `src/test/scala/dispatcher/services/outbound/internal/config/ConfigRetrieverTest.scala`
- `src/test/scala/dispatcher/services/outbound/internal/config/ConfigMessagePlayServiceTest.scala`

This suggests Dispatcher consumes or retrieves configuration that affects routing and event behavior.

Configuration likely controls:

- event visibility
- frontend/vehicle flags
- notification eligibility
- reset behavior
- source-system routing
- TTL-related rules
- event pattern matching

Spring Boot equivalent:

- `ConfigConsumer`
- `RoutingConfigService`
- local in-memory config cache
- compacted Kafka config topic

Interview phrase:

> We externalized event behavior into configuration. Dispatcher used config to decide whether an event should be routed, whether it can trigger notifications, and how it should be handled by channel.

---

## Notifications

- `src/main/scala/dispatcher/services/outbound/NotificationMsgProcessor.scala`
- `src/main/scala/dispatcher/services/outbound/fns/FnsRequestService.scala`
- `src/main/scala/dispatcher/services/outbound/fns/FrontendNotificationActorTest.scala`
- `src/main/scala/dispatcher/services/outbound/mnp/FrontendNotificationConsumer.scala`
- `src/main/scala/dispatcher/services/outbound/mnp/FrontendNotificationEventProcessor.scala`
- `src/main/scala/dispatcher/services/outbound/mnp/FrontendNotifier.scala`
- `src/main/scala/dispatcher/services/outbound/mnp/MnpKafkaProducerService.scala`
- `src/main/scala/dispatcher/services/outbound/mnp/MnpRequestService.scala`

Dispatcher is also responsible for push/customer notification flow.

Flow:

```text notification-flow.txt
Supplier stores important NOK event
        |
        | publishes notification message
        v
Kafka: FNS_Notification / MNP notification topic
        |
        v
Dispatcher notification consumer
        |
        | checks config: notifyPortal, criticality, brand, country
        v
Calls FNS/MNP endpoint
        |
        v
Customer receives push notification
```

Interview phrase:

> Supplier owns the event state, but Dispatcher owns the communication routing to notification systems. That keeps notification policy separate from persistence.

---

## Reset forwarding

- `src/main/scala/dispatcher/services/AcdcResetTracker.scala`
- `src/main/scala/dispatcher/services/outbound/acdc/AcdcResetEventService.scala`
- `src/main/scala/dispatcher/services/outbound/acdc/AcdcResetKafkaProducerService.scala`
- `src/main/scala/dispatcher/services/outbound/acdc/ResetMsgProcessorHandler.scala`
- `src/test/scala/dispatcher/services/outbound/acdc/AcdcResetKafkaProducerServiceTest.scala`
- `src/test/scala/dispatcher/services/outbound/acdc/AcdcResetMsgProcessorHandlerTest.scala`
- `src/test/scala/dispatcher/services/outbound/acdc/AcdcResetTrackerTest.scala`

Reset means a frontend/vehicle action wants to reset or clear an event.

Flow:

```text reset-flow.txt
Customer/app calls Supplier reset API
        |
        v
Supplier validates request
        |
        | publishes reset request to Kafka
        v
Dispatcher consumes reset request
        |
        | maps event/source to correct upstream reset topic
        v
Publishes reset trigger to source system
        |
        v
Source system later emits updated OK/reset event
        |
        v
Dispatcher -> Supplier -> Cassandra updated
```

Interview phrase:

> Reset is asynchronous. The API returns quickly after publishing a reset request, and Dispatcher handles routing that request to the correct upstream source system.

---

## Dispatcher health

- `src/main/scala/dispatcher/services/HealthcheckService.scala`
- `src/main/twirl/HealthcheckTemplates.scala.html`
- `src/main/twirl/CheckTemplate.scala.html`
- `src/main/twirl/SubServiceCheckTemplate.scala.html`
- `src/main/scala/dispatcher/controllers/IsAliveModel.scala`

Spring Boot equivalent:

- `/actuator/health`
- `/isAlive`
- `/check`
- custom Kafka/HTTP dependency health indicators

---

# 4. Dispatcher Processing Flow

This is the core pipeline you should remember.

```text dispatcher-processing.txt
Raw Kafka message arrives from upstream topic
        |
        v
Deserialize envelope
        |
        | failure
        v
Error topic / DLQ

If success:
        |
        v
Validate required fields
VIN/PVIN, brand, event id, payload
        |
        | failure
        v
Error topic / DLQ

If success:
        |
        v
Normalize data
brand code, event type, metadata, additional info
        |
        v
Resolve vehicle identity
PVIN -> VIN / region / market
        |
        | cache hit
        | cache miss -> external VDS/VWAC call
        v
Apply routing/config rules
showInFrontend, showInVehicle, notifyPortal, reset config
        |
        v
Publish normalized event to internal service_event topic
key = VIN
        |
        v
Commit Kafka offset manually
```

The key reliability point:

> Dispatcher should only commit its Kafka offset after the normalized event has been successfully produced to the internal topic. Otherwise, a failure between consume and produce could lose the event.

---

# 5. `sto-supplier-service`

## 5.1 What Supplier Does

Supplier is the **state owner and API layer**.

It consumes normalized events from Dispatcher, stores them in Cassandra, and exposes REST APIs for consumers.

Its responsibilities are:

1. Consume normalized internal events.
2. Convert them into canonical domain models.
3. Check existing Cassandra state.
4. Apply idempotency and timestamp ordering.
5. Insert/update/delete Cassandra records with TTL.
6. Publish notification trigger events if needed.
7. Serve frontend, vehicle, external, config, reset, and GDPR APIs.
8. Enrich API responses using CMS/zFDI, recall, ORU, and other services.
9. Apply authentication and authorization.
10. Provide health and dependency checks.

---

## 5.2 Supplier File Map

## Startup / shared service

- `src/main/scala/supplier/services/SupplierStartService.scala`
- `src/main/scala/supplier/services/ApplicationSharedService.scala`
- `src/main/scala/supplier/services/HealthcheckService.scala`

Spring Boot equivalent:

- application startup runner
- dependency initialization
- actuator health indicators

---

## Kafka consumer path

- `src/main/scala/supplier/services/incoming/ServiceEventConsumer.scala`
- `src/main/scala/supplier/services/EventMsgProcessor.scala`
- `src/main/scala/supplier/services/incoming/KafkaConsumerCheckService.scala`
- `src/main/scala/supplier/services/incoming/ProducerCheckService.scala`
- `src/main/scala/supplier/services/incoming/ConfigMessagePlayService.scala`

These are the writer-mode components.

Spring Boot equivalent:

- `@KafkaListener(topics = "service_event")`
- `ServiceEventProcessor`
- manual `Acknowledgment`
- producer health check
- config topic consumer

Interview phrase:

> Supplier listens to the internal service event topic. For each event, it checks whether the incoming state is newer than what Cassandra already has, writes only meaningful updates, and commits the Kafka offset only after persistence succeeds.

---

## Cassandra ownership

- `src/main/scala/supplier/services/CassandraConnectorService.scala`
- `src/test/scala/supplier/services/CassandraConnectorServiceTest.scala`
- `src/main/resources/cassandra.conf`
- `src/universal/script/connectToCassandra.sh`
- `src/universal/script/update-ttl.sh`

Supplier owns Cassandra access.

It likely writes and reads:

- main service events
- QCI events
- OCC/recall cache
- configuration/fallback data
- DSGVO-related data

Spring Boot equivalent:

- Spring Data Cassandra repository
- `CassandraTemplate`
- DAO layer
- TTL-based writes

Interview phrase:

> Cassandra was modeled around the main query pattern: get all active events for a VIN. So the partitioning is VIN-centric, which gives fast reads for app and vehicle requests.

---

## CMS/zFDI enrichment

- `src/main/scala/supplier/services/CmsDataService.scala`
- `src/main/scala/supplier/services/cms/CmsCache.scala`
- `src/main/scala/supplier/controllers/model/CmsDataProviderFactory.scala`
- `src/main/scala/supplier/controllers/action/DetailTextsController.scala`
- `src/main/resources/cms/languages.json`
- `src/main/resources/cms/warnings/*.json`

Supplier enriches events with localized text.

Example:

An event may contain:

- `txtId = W042`
- `picId = P123`
- language/country from request

Supplier maps that to localized text:

- German: “Reifendruck prüfen”
- English: “Check tire pressure”
- French: equivalent French message

Interview phrase:

> The event payload does not carry full localized customer-facing text. It carries references like text IDs. Supplier resolves those using CMS/zFDI data and returns localized responses based on the requested language and country.

---

## API services

- `src/main/scala/supplier/http4s/services/ConfigurationService.scala`
- `src/main/scala/supplier/http4s/services/IbdService.scala`
- `src/main/scala/supplier/http4s/services/MbbTokenService.scala`
- `src/main/scala/supplier/http4s/services/MsgService.scala`
- `src/main/scala/supplier/http4s/services/ResponseUtil.scala`

Spring Boot equivalent:

- service layer below REST controllers

---

## API routes

- `src/main/scala/supplier/http4s/routes/BuildResponseRoutes.scala`
- `src/main/scala/supplier/http4s/routes/ConfigurationRoutes.scala`
- `src/main/scala/supplier/http4s/routes/DsgvoRoutes.scala`
- `src/main/scala/supplier/http4s/routes/IbdRoutes.scala`
- `src/main/scala/supplier/http4s/routes/MsgRequestRoutes.scala`
- `src/main/scala/supplier/http4s/routes/ResetRoutes.scala`
- `src/main/scala/supplier/http4s/routes/ServicePortletRoutes.scala`
- `src/main/scala/supplier/http4s/routes/SupplierRoutes.scala`
- `src/main/scala/supplier/http4s/routes/QueryParamsUtils.scala`
- `src/main/scala/supplier/http4s/routes/RoutesUtils.scala`

Spring Boot equivalent:

- `FrontendEventController`
- `VehicleEventController`
- `ServicePortletController`
- `ResetController`
- `DsgvoController`
- `ConfigurationController`
- `HealthController`

---

## Authentication / authorization

- `src/main/scala/supplier/http4s/middleware/auth/DsgvoAuthMiddleware.scala`
- `src/main/scala/supplier/http4s/middleware/auth/MsgBasicAuthMiddleware.scala`
- `src/main/scala/supplier/http4s/middleware/auth/StoAuthentication.scala`
- `src/main/scala/supplier/http4s/middleware/auth/TokenAuthMiddleware.scala`
- `src/main/scala/supplier/config/OAuthConfig.scala`
- `src/main/resources/authTokenCheck.conf`
- `src/main/resources/cloudIdp/cloudidp.conf`

Spring Boot equivalent:

- Spring Security filters
- OAuth2/JWT validation
- route-specific authorization
- scopes/roles
- basic auth for selected routes

Interview phrase:

> Supplier enforced route-specific security. Frontend, vehicle, external partner, configuration, reset, and DSGVO routes had different authentication and authorization requirements.

---

## Refiner/enrichment chain

- `src/main/scala/supplier/http4s/middleware/refiner/RefinerMiddleware.scala`
- `src/main/scala/supplier/http4s/middleware/refiner/LoggingRefiner.scala`
- `src/main/scala/supplier/http4s/middleware/refiner/eventrefiner/EventRefinerMiddleware.scala`
- `src/main/scala/supplier/http4s/middleware/refiner/orurefiner/OruRefinerHandler.scala`
- `src/main/scala/supplier/http4s/middleware/refiner/orurefiner/OruRefinerMiddleware.scala`
- `src/main/scala/supplier/http4s/middleware/refiner/orurefiner/OruResponseMapper.scala`
- `src/main/scala/supplier/http4s/middleware/refiner/occrefiner/OccDataRefinerHandler.scala`
- `src/main/scala/supplier/http4s/middleware/refiner/occrefiner/OccRefinerMiddleware.scala`

This is important for advanced interview depth.

Supplier does not just return raw Cassandra rows. For some APIs, especially OCC-style responses, it builds a response by applying multiple refiners.

Possible refiners:

1. STO event refiner — Cassandra service events.
2. QCI refiner — quality/customer information.
3. OCC/recall refiner — recall campaign information.
4. ORU refiner — online update status.
5. CMS/zFDI refiner — localized text.
6. Filtering/refinement — frontend/vehicle visibility, priority, include params.

Interview phrase:

> The Supplier API used a refiner-style pipeline. Each refiner contributed or transformed part of the response: STO events from Cassandra, recall data from external services, ORU status, CMS localization, filtering, and ordering.

---

## OCC and response models

- `src/main/scala/supplier/services/model/OccResponse.scala`
- `src/main/scala/supplier/controllers/model/OnlineCarCareResponse.scala`
- `src/main/scala/supplier/config/OccResponseFilter.scala`
- `src/main/scala/supplier/config/RecallTypeMapper.scala`
- `src/main/resources/occ/prio.conf`
- `src/test/resources/oruResponseMap.conf`
- `src/test/resources/oruResponseMapOruRelevantCampaigns.conf`

This area handles response shaping, filtering, and priority.

Interview phrase:

> OCC responses were not direct database dumps. They were business responses with filtering, deduplication, ordering, campaign mapping, and localization.

---

## Reset

- `src/main/scala/supplier/http4s/routes/ResetRoutes.scala`
- `src/main/scala/supplier/config/KafkaResetProducerConfig.scala`
- `src/main/scala/supplier/actors/KafkaResetProducer.scala`
- `src/test/scala/supplier/kafka/KafkaResetProducerTest.scala`
- `src/test/scala/supplier/http4s/routes/ResetRoutesTest.scala`

Supplier accepts reset API calls but does not directly reset the upstream source. It publishes a reset request to Kafka.

Interview phrase:

> Supplier exposes the reset API, validates that the event is resettable, then publishes a reset command to Kafka. Dispatcher later routes that command to the appropriate upstream system.

---

## DSGVO/GDPR

- `src/main/scala/supplier/http4s/routes/DsgvoRoutes.scala`
- `src/main/scala/supplier/model/Dsgvo.scala`
- `src/test/scala/supplier/services/DsgvoConsumerActorTest.scala`
- `src/test/scala/supplier/services/DsgvoPlayServiceTest.scala`
- `src/test/resources/dsgvo/request-1.json`
- `src/test/resources/dsgvo/request-2.json`
- `src/test/resources/dsgvo/response-1.json`
- `src/main/resources/routine/ksuDeletion.conf`
- `src/universal/script/ksu-deletion-main.sh`
- `src/universal/script/ksu-deletion-service.sh`

Supplier supports privacy deletion flows.

Business meaning:

> Since VIN-related event data may be personal or legally sensitive, Supplier provides inquiry and deletion flows for DSGVO/GDPR compliance.

Spring Boot equivalent:

- `DsgvoController`
- `DsgvoService`
- scheduled cleanup job
- Kafka consumer for deletion events
- Cassandra delete operations

---

## Trace/logging

- `src/main/scala/supplier/utility/trace/TraceId.scala`
- `src/main/scala/supplier/utility/trace/Tracing.scala`
- `src/main/scala/supplier/utility/trace/Rich.scala`
- `src/main/scala/supplier/services/logacc/TraceHeaders.scala`
- `src/main/scala/supplier/services/logacc/TraceLogging.scala`
- `src/main/scala/supplier/http4s/middleware/LogAccMiddleware.scala`

Interview phrase:

> We propagated trace context across REST and Kafka so an event could be followed end-to-end by VIN, event ID, and trace ID.

---

# 6. Supplier Writer Flow

This is the Kafka-to-Cassandra flow.

```text supplier-writer-flow.txt
Normalized service event arrives from Kafka
        |
        v
Supplier consumes service_event
        |
        v
Deserialize into canonical domain model
        |
        v
Derive vehicle context
VIN, brand, region, country, event type
        |
        v
Read existing Cassandra record by VIN + modelId/eventId
        |
        v
Apply idempotency logic
        |
        +-- no existing record -> insert
        |
        +-- incoming newer -> update
        |
        +-- incoming older -> ignore stale event
        |
        +-- same timestamp + meaningful status change -> update
        |
        +-- duplicate/no change -> ignore
        |
        v
Write Cassandra with appropriate TTL if needed
        |
        v
If NOK and notification eligible:
publish notification trigger event
        |
        v
Commit Kafka offset manually
```

Most important interview phrase:

> Supplier gives us business-level exactly-once behavior on top of Kafka’s at-least-once delivery. Kafka may redeliver messages, but the Cassandra idempotency check ensures the final state is correct.

---

# 7. Supplier Reader/API Flow

This is the REST API side.

```text supplier-reader-flow.txt
Client calls Supplier API
myAudi / vehicle / workshop / admin
        |
        v
Authentication middleware
JWT/basic/auth token/scope validation
        |
        v
Build request context
VIN, brand, region, language, channel, API version
        |
        v
Query Cassandra for VIN-specific event state
        |
        v
Apply business filters
showInFrontend / showInVehicle / include params / event type
        |
        v
Optional enrichment
CMS/zFDI text, recall/OCC data, ORU status
        |
        v
Map to response DTO
        |
        v
Return HTTP response
200 if complete
206 if partial data available
4xx if auth/request issue
5xx if unrecoverable internal issue
```

---

# 8. The Most Important Supplier Concept: Idempotency

This is probably the deepest technical talking point.

Kafka gives **at-least-once delivery**, not exactly-once business state.

So Supplier must assume:

- the same event can be delivered twice
- events can arrive late
- upstream can send duplicates
- retries can happen
- service restarts can reprocess messages

The solution is timestamp-gated Cassandra writes.

```text idempotency.txt
Incoming event: VIN + modelId + timestamp + status

Check Cassandra:

1. No existing row
   -> Insert

2. Existing row found, incoming timestamp is newer
   -> Update

3. Existing row found, incoming timestamp is older
   -> Discard stale event

4. Timestamp is equal/close, but status or criticality changed
   -> Update using tolerance rule

5. Timestamp is equal/close, no meaningful change
   -> Ignore duplicate

6. Status is PASSIVE
   -> Mark inactive / apply short TTL
```

Interview wording:

> The Kafka offset is committed only after the Cassandra write succeeds. If Cassandra is unavailable, we do not commit. Kafka redelivers later. Because the write logic is idempotent, reprocessing is safe.

This is very strong.

---

# 9. Cassandra Model

You do not need to know every table column, but you should understand the modeling philosophy.

## Cassandra is used as a current-state store

Not a full history/event log.

Main query:

> “Give me all current active events for this VIN.”

So Cassandra is modeled around VIN lookup.

Likely tables/concepts:

| Table/concept | Purpose |
|---|---|
| `events_audi` | Current Audi event state by VIN |
| `events_porsche` | Current Porsche event state by VIN |
| `qci_events_audi` | QCI event state |
| `occ_events_audi` | Cached OCC/recall response data |
| config/fallback tables | configuration/fallback support |
| DSGVO deletion data | privacy-related cleanup |

Common key idea:

```text cassandra-model.txt
Partition key: VIN
Clustering key: modelId/eventId/language depending on table

Read pattern:
SELECT all current events WHERE vin = ?

Write pattern:
UPSERT latest event state with TTL
```

Interview phrase:

> Cassandra was a good fit because the dominant query pattern was VIN-based lookup, and the system needed high write throughput, high availability, and TTL-based lifecycle management.

---

# 10. TTL Strategy

TTL is another important design point.

Supplier writes events with TTL depending on status/config.

| Event state | TTL idea | Why |
|---|---|---|
| NOK / active | Longer TTL | customer still needs to see/action it |
| OK / resolved | Medium TTL | keep briefly for consistency/UI |
| PASSIVE / hidden | Short TTL | remove quickly |
| GDPR deletion | explicit delete | legal requirement |

Interview phrase:

> Normal lifecycle cleanup was handled by Cassandra TTL at write time. Legal/privacy deletion was handled explicitly through DSGVO/KSU deletion flows.

---

# 11. API Categories in Supplier

## 11.1 Frontend / MSG API

Likely around:

- `MsgRequestRoutes.scala`
- `MsgService.scala`
- `BuildResponseRoutes.scala`
- `BuildResponseJsonAction.scala`
- `OnlineCarCareResponse.scala`

Used by:

- myAudi app
- web portal

Applies:

- `showInFrontend`
- localization
- customer-facing mapping
- authentication

---

## 11.2 IBD / Vehicle API

Likely around:

- `IbdRoutes.scala`
- `IbdService.scala`

Used by:

- in-vehicle display/app systems

Applies:

- `showInVehicle`
- vehicle-specific auth/context
- possibly ignition-time sync

---

## 11.3 Service Portlet / External API

Likely around:

- `ServicePortletRoutes.scala`

Used by:

- workshops
- partner systems
- service advisors

Applies:

- partner/system auth
- VIN-specific event lookup
- workshop-oriented response

---

## 11.4 Reset API

Likely around:

- `ResetRoutes.scala`
- `KafkaResetProducer.scala`

Used by:

- frontend/vehicle to reset events

Does:

- validate reset permission
- publish reset request
- return async response

---

## 11.5 Configuration API

Likely around:

- `ConfigurationRoutes.scala`
- `ConfigurationService.scala`
- `ConfigInitializationService.scala`
- `ConfigRetrieverProducer.scala`
- `eventConfig/eventDefault.conf`
- `eventConfig/initialEventConfig.conf`

Used by:

- internal tooling
- initial setup
- event configuration management

---

## 11.6 DSGVO API

Likely around:

- `DsgvoRoutes.scala`
- `Dsgvo.scala`

Used by:

- compliance systems
- deletion/inquiry flows

---

# 12. The Core End-to-End Happy Path

This is the story to know cold.

```text happy-path.txt
1. Upstream ACDC publishes raw NOK event to Kafka.

2. Dispatcher consumes it.
   - deserializes
   - validates
   - normalizes brand/event fields
   - resolves PVIN to VIN
   - applies config
   - publishes normalized event to service_event topic

3. Supplier consumes service_event.
   - checks Cassandra for existing VIN + modelId
   - no existing row, so inserts event with NOK status
   - applies TTL
   - publishes notification trigger if eligible
   - commits Kafka offset

4. Dispatcher consumes notification trigger.
   - checks notify config
   - calls FNS/MNP endpoint

5. Customer receives push notification.

6. Customer opens myAudi app.
   - app calls Supplier API
   - Supplier validates token
   - queries Cassandra by VIN
   - enriches and filters response
   - returns active event list
```

One-liner:

> A raw event becomes a customer-visible notification through Dispatcher normalization, Supplier persistence, Cassandra state, and Supplier APIs.

---

# 13. Reset Flow

```text reset-path.txt
1. Customer resets event in app.

2. App calls Supplier reset endpoint.

3. Supplier:
   - authenticates user
   - verifies event exists
   - verifies event is resettable
   - publishes reset request to Kafka
   - returns HTTP 202 Accepted

4. Dispatcher consumes reset request.

5. Dispatcher:
   - maps event type to source system
   - publishes reset trigger to source-specific Kafka topic

6. Source system processes reset.

7. Later source emits OK/reset event.

8. Dispatcher normalizes it.

9. Supplier updates Cassandra.

10. Event disappears or changes state in customer API response.
```

Interview phrase:

> Reset is asynchronous because the user-facing API should not wait for upstream systems. Supplier accepts the command, Dispatcher routes it, and eventual state comes back through the normal event pipeline.

---

# 14. OCC / Enriched Response Flow

This is the advanced API flow.

```text occ-flow.txt
Client calls OCC/service event API for VIN
        |
        v
Supplier authenticates request
        |
        v
Build request context:
VIN, brand, region, language, version, include params
        |
        v
Run response/refiner pipeline:
        |
        +--> STO events from Cassandra
        |
        +--> QCI events from Cassandra
        |
        +--> OCC/recall data from cache or external recall API
        |
        +--> ORU status enrichment
        |
        +--> CMS/zFDI localized text enrichment
        |
        v
Apply filtering:
showInFrontend, include params, priority, deduplication
        |
        v
Return response:
200 if complete
206 if partial external data missing
```

Interview phrase:

> The Supplier response layer was resilient. If an external enrichment service like recall was unavailable, we could still return the locally stored STO/QCI data instead of failing the whole request.

---

# 15. Error Handling Philosophy

## Dispatcher errors

| Failure | Handling |
|---|---|
| Bad raw payload | DLQ/error topic |
| missing required fields | DLQ/error topic |
| PVIN/VIN resolution failure | retry, then DLQ |
| external notification failure | retry/DLQ/logging depending on flow |
| internal publish failure | do not commit offset |

## Supplier errors

| Failure | Handling |
|---|---|
| Cassandra unavailable | do not commit Kafka offset |
| duplicate/stale event | ignore and commit |
| external enrichment unavailable | partial response or cached response |
| bad API auth | 401/403 |
| invalid reset/config request | 400/404/409 style response |

Important distinction:

> Bad messages are parked and committed so they don’t block the pipeline. Infrastructure failures are not committed so Kafka can retry later.

---

# 16. Spring Boot Translation

Since you want to present this as Spring Boot, map it like this.

## Dispatcher as Spring Boot

| Actual Scala/http4s concept | Spring Boot equivalent |
|---|---|
| Kafka consumer processors | `@KafkaListener` |
| Kafka producers | `KafkaTemplate` |
| VDS/VWAC clients | `WebClient` clients |
| Config services | `@Service` with in-memory cache |
| HealthcheckService | Actuator `HealthIndicator` |
| Reset processors | Kafka listener + producer service |
| Notification processors | Kafka listener + WebClient service |

Possible Spring-style package layout:

```text spring-dispatcher-layout.txt
com.audi.sto.dispatcher
  config
    KafkaConsumerConfig
    KafkaProducerConfig
    WebClientConfig
    CacheConfig
  consumer
    AcdcEventConsumer
    QciEventConsumer
    NotificationConsumer
    ResetConsumer
  service
    EventNormalizationService
    VinResolutionService
    RoutingConfigService
    NotificationDispatchService
    ResetForwardingService
  client
    VdsClient
    VwacClient
    FnsClient
  producer
    ServiceEventProducer
    ErrorTopicProducer
  health
    KafkaHealthIndicator
    ExternalDependencyHealthIndicator
```

## Supplier as Spring Boot

| Actual Scala/http4s concept | Spring Boot equivalent |
|---|---|
| Routes | `@RestController` |
| Middleware auth | Spring Security filter chain |
| CassandraConnectorService | Spring Data Cassandra repository/service |
| ServiceEventConsumer | `@KafkaListener` |
| RefinerMiddleware | service pipeline / chain of responsibility |
| CmsDataService | CMS service/cache |
| DsgvoRoutes | `DsgvoController` |
| ResetRoutes | `ResetController` |
| HealthcheckService | Actuator `HealthIndicator` |

Possible Spring-style package layout:

```text spring-supplier-layout.txt
com.audi.sto.supplier
  config
    SecurityConfig
    KafkaConfig
    CassandraConfig
    WebClientConfig
  consumer
    ServiceEventConsumer
    ConfigConsumer
    DsgvoConsumer
  controller
    FrontendEventController
    VehicleEventController
    ServicePortletController
    ResetController
    ConfigurationController
    DsgvoController
    HealthController
  service
    ServiceEventProcessingService
    IdempotencyService
    CassandraEventService
    CmsDataService
    OccResponseService
    ResetService
    ConfigurationService
    DsgvoService
  refiner
    StoEventRefiner
    QciRefiner
    OccRefiner
    OruRefiner
    LocalizationRefiner
  repository
    ServiceEventRepository
    QciEventRepository
    OccEventRepository
  client
    RecallClient
    OruClient
    CloudIdpClient
    CmsClient
```

---

# 17. Interview Answer: “Explain the Two Services”

You can say:

> STO had two main backend services: Dispatcher and Supplier. Dispatcher was the ingestion service. It consumed raw upstream Kafka events from systems like ACDC, predictive maintenance, QCI, and regional feeds. It deserialized and validated the messages, normalized brand and event data, resolved PVIN to VIN using external vehicle services with caching, applied dynamic routing configuration, and published a clean normalized event to an internal Kafka topic.
>
> Supplier was the state and API service. It consumed those normalized events, performed idempotency checks against Cassandra, and stored the latest vehicle-event state per VIN. It exposed REST APIs for the myAudi app, in-vehicle systems, workshop/service-portlet integrations, reset flows, configuration, and GDPR deletion. On read paths it enriched the stored events with localized CMS text, recall campaign data, and ORU update status.
>
> Kafka separated the two services, which gave us independent scaling, buffering, retry, and deployment. Cassandra was chosen because our primary query was VIN-based: “give me all current events for this vehicle.” The most important reliability design was manual Kafka offset handling combined with idempotent Cassandra writes, so duplicate or stale Kafka messages could not corrupt the current vehicle state.

---

# 18. If Interviewer Asks: “What Was the Hardest Part?”

Best answer:

> The hardest part was correctness under duplicate and out-of-order events. Kafka gives at-least-once delivery, so the Supplier had to be idempotent. We solved it by reading the existing Cassandra record for the same VIN and event ID, comparing timestamps and status, and only writing if the incoming event was newer or meaningfully different. We committed Kafka offsets only after successful writes. This gave us business-level exactly-once behavior without distributed transactions.

---

# 19. If Interviewer Asks: “Why Kafka?”

Answer:

> Kafka decoupled ingestion from persistence and API serving. Dispatcher could continue consuming raw upstream events even if Supplier was temporarily down. Supplier could catch up later from Kafka offsets. Kafka also gave partition-based parallelism, retry/replay capability, error-topic handling, and clean service boundaries.

---

# 20. If Interviewer Asks: “Why Cassandra?”

Answer:

> Cassandra matched our access pattern. The main query was always VIN-based: fetch all current events for a specific vehicle. We did not need joins or complex relational queries. We needed high write throughput, high availability, horizontal scalability, and TTL-based expiry. Cassandra was ideal for that.

---

# 21. If Interviewer Asks: “How Do You Avoid Duplicate Notifications?”

Answer:

> Supplier only emits notification triggers after the idempotency decision says the event produced a meaningful state change. If Kafka redelivers the same event, Supplier sees that the Cassandra state already reflects it and skips both the write and the notification trigger.

---

# 22. If Interviewer Asks: “Who Owns Reset?”

Answer:

> Supplier owns the customer-facing reset API. Dispatcher owns reset routing to upstream systems. The customer calls Supplier, Supplier validates and publishes a reset request to Kafka, and Dispatcher maps that request to the correct source-system reset topic. This keeps the API clean and hides upstream complexity from clients.

---

# 23. If Interviewer Asks: “Who Owns Notification?”

Answer:

> Supplier decides that an important state change occurred and publishes a notification trigger. Dispatcher handles the actual notification dispatch because it owns routing and outbound communication policy. That separation prevents persistence logic from being tightly coupled to FNS/MNP integration.

---

# 24. Final Mental Model

Remember this:

```text final-model.txt
Dispatcher:
"Can I understand this raw event, identify the vehicle, and route it?"

Supplier:
"Is this event newer than what I already know, should I store it, and how should I show it to consumers?"
```

Or even shorter:

> **Dispatcher cleans and routes. Supplier stores and serves. Kafka connects. Cassandra remembers.**