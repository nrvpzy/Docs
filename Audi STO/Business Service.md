# Audi STO — Business Deep Dive Report

## 1. Executive Summary

Audi STO, or **Service Technik Online**, is a backend platform designed to digitalize and modernize the way Audi handles **vehicle service information, recalls, warnings, predictive maintenance events, and customer-facing service communication**.

At a business level, STO answers one core question:

> **For this specific vehicle, identified by its VIN, what service-relevant information should the customer, vehicle, workshop, or partner system know right now?**

Before systems like STO, many service-related communications were reactive. A customer might only discover a recall when they visited a workshop, received a physical letter, or saw a warning in the vehicle. STO changes that by moving this communication earlier, making it digital, personalized, and available across multiple channels such as:

- myAudi mobile app
- myAudi web portal
- in-vehicle systems
- workshop/service partner systems
- internal Audi operational systems
- compliance and data protection systems

The platform is built around a **Dispatcher-Supplier architecture**.

- The **Dispatcher** receives raw vehicle events from upstream systems, understands them, identifies the correct vehicle, and routes clean events into the STO platform.
- The **Supplier** stores the current state of each vehicle’s service events and serves that information through APIs to customers, vehicles, workshops, and internal systems.

In simple terms:

> **Dispatcher prepares the information. Supplier owns and delivers the information.**

---

# 2. Business Problem STO Solves

## 2.1 The Old World: Reactive Service Communication

Historically, vehicle service communication was often delayed and fragmented.

A customer might only learn about a vehicle issue through:

1. A warning lamp in the car.
2. A workshop diagnosis.
3. A recall letter sent by post.
4. A service advisor checking systems during a workshop appointment.
5. Separate regional systems or brand-specific communication channels.

This created several business issues.

---

## 2.2 Late Customer Awareness

If a vehicle had an open recall, pending service campaign, or predictive maintenance issue, the customer often found out too late.

For example:

> A recall campaign may already exist in Audi’s backend systems, but unless the customer receives a letter or visits a workshop, they may remain unaware.

That is a poor customer experience, especially for safety-related or time-sensitive issues.

STO solves this by making these events digitally available and deliverable to the customer much earlier.

---

## 2.3 Inefficient Workshop Planning

Workshops and service partners need accurate information before a customer arrives.

Without STO, a workshop may only identify relevant service campaigns after the vehicle is already in the workshop.

This can cause:

- Longer appointment times
- Missed campaign opportunities
- Parts not being available
- Poor service advisor preparation
- Lower customer satisfaction

STO gives workshops better visibility into what is relevant for a specific VIN before or during the service journey.

---

## 2.4 Fragmented Vehicle Event Data

Vehicle-related information comes from many different upstream systems:

- Recall systems
- Service campaign systems
- Predictive maintenance systems
- Vehicle diagnostic/event systems
- Regional OEM feeds
- Quality/customer information systems
- Online update status systems

Each system may send information in a different format, with different identifiers, different regions, and different business meanings.

STO acts as a central platform that normalizes and consolidates this information into a single VIN-centric view.

---

## 2.5 Channel-Specific Communication Complexity

Not every event should be shown everywhere.

Some events should appear in the myAudi app.

Some should appear in the vehicle.

Some should only be visible to workshops.

Some should trigger a push notification.

Some should be hidden after resolution.

Some should be market-specific or brand-specific.

STO provides a configurable way to decide:

- Which events are shown to customers
- Which events are shown in the vehicle
- Which events trigger notifications
- Which events are available to workshops
- Which events expire after a certain period
- Which events can be reset by the customer

This allows Audi to manage communication policy without treating every event the same way.

---

# 3. Business Purpose of STO

At a business level, STO exists to:

> **Turn raw vehicle and campaign signals into personalized, channel-appropriate, VIN-specific service information.**

That means STO takes information from upstream systems and transforms it into something usable by:

- Customers
- Vehicles
- Workshops
- Partner systems
- Internal Audi teams
- Compliance systems

The platform is not just a data pipe. It is a **business decision and delivery platform** for service-related vehicle communication.

---

# 4. Main Business Capabilities

## 4.1 VIN-Centric Vehicle Event Management

The central business object in STO is the **vehicle**, identified by its VIN.

Everything revolves around the question:

> What is currently relevant for this VIN?

Examples:

- Does this vehicle have an open recall?
- Does this vehicle have a predictive maintenance warning?
- Does this vehicle have a quality campaign?
- Should this event be shown to the customer?
- Is this event visible in the vehicle?
- Has this event already been resolved?
- Should a push notification be sent?
- Is the event still active or expired?
- Is the customer allowed to reset it?

STO maintains a current view of vehicle service state.

---

## 4.2 Recall and Campaign Visibility

One of the key use cases is making recall and service campaign information visible digitally.

Instead of relying only on postal letters or workshop discovery, STO enables recall/campaign information to appear in digital channels.

Customer-facing example:

> A customer opens the myAudi app and sees that their vehicle has an open recall or service campaign, along with localized explanatory text and recommended next steps.

Workshop-facing example:

> A service advisor checks the customer’s VIN and sees open campaign information before the appointment.

Business value:

- Better safety communication
- Higher recall completion rate
- Better workshop readiness
- Better customer trust

---

## 4.3 Predictive Maintenance and Warning Events

STO also supports events related to warnings or predictive maintenance.

These may include signals such as:

- Vehicle warning states
- Maintenance reminders
- Predictive component issues
- Service-relevant technical alerts

The goal is to inform the customer or vehicle systems before the problem becomes serious.

Business value:

- Earlier intervention
- Reduced breakdown risk
- Better customer experience
- Higher service retention

---

## 4.4 Multi-Channel Delivery

STO serves different channels, each with a different business purpose.

## Customer App / Portal

The myAudi app and portal need customer-friendly information.

This means the response must be:

- VIN-specific
- Customer-readable
- Localized
- Filtered for frontend visibility
- Action-oriented

The customer should not see internal technical noise.

They should see something like:

> “Your vehicle has an open service campaign. Please contact your Audi partner.”

---

## In-Vehicle Systems

The vehicle channel has a different purpose.

In-vehicle systems may display relevant service or warning messages directly inside the car.

This is especially important because the vehicle is where the customer experiences the issue.

The vehicle channel may be used for:

- Ignition-time synchronization
- Dashboard or infotainment display
- Vehicle-specific warnings
- Service messages that should be shown while using the car

Not every frontend event belongs in the vehicle, and not every vehicle event belongs in the app. STO supports this separation.

---

## Workshop and Service Portlet

Workshops need a more operational view.

They care about:

- Which campaigns are open?
- What is the current status?
- What action should be taken?
- Is this event relevant for service planning?
- Is there a recall or quality campaign linked to the vehicle?

STO gives service partners a VIN-specific view that supports workshop readiness.

---

## Internal Systems

Internal Audi systems may use STO for:

- Operational monitoring
- Configuration management
- Compliance processes
- Data deletion workflows
- Regional or market-specific governance

STO therefore supports both customer-facing and internal enterprise use cases.

---

# 5. Stakeholders and Users

## 5.1 Vehicle Owners / Customers

Customers are the most visible beneficiaries.

They receive timely, personalized information in the myAudi app, portal, or in-vehicle display.

For them, STO improves:

- Awareness
- Safety
- Trust
- Convenience
- Service planning

Instead of waiting for workshop diagnosis or postal communication, the customer gets digital information directly.

---

## 5.2 Workshops and Service Partners

Workshops benefit because they can better prepare for incoming vehicles.

They can see:

- Open service events
- Recall/campaign status
- Relevant vehicle service information
- Actions needed for a VIN

This improves appointment quality and reduces surprises.

---

## 5.3 Audi Market and Regional Teams

Markets and regions may have different legal, operational, and communication requirements.

For example:

- A campaign may be relevant in one region but not another.
- Visibility rules may differ by market.
- Push notification behavior may vary.
- Vehicle generations may require different handling.

STO supports market-specific and region-specific configuration.

---

## 5.4 Internal Compliance Teams

Because VIN-related data can be sensitive, STO supports GDPR/DSGVO workflows.

Compliance teams need the ability to:

- Inquire what data is stored for a vehicle/customer context
- Delete data when legally required
- Ensure retention rules are followed

STO includes business support for these privacy requirements.

---

## 5.5 Downstream Digital Products

STO supports digital products such as:

- myAudi app
- Audi portal
- in-vehicle digital services
- service partner portals
- workshop integrations

These products depend on STO to provide clean, current, vehicle-specific service data.

---

# 6. High-Level Architecture

At a high level, STO has four major layers:

```text
1. Upstream Event Sources
2. Dispatcher Service
3. Supplier Service
4. Consumer Channels
```

A simplified architecture:

```text
Upstream Systems
Recall / ACDC / Predictive / QCI / Regional Feeds
        |
        v
Dispatcher Service
Ingestion, validation, vehicle identification, routing
        |
        v
Internal Event Stream
Clean normalized vehicle events
        |
        v
Supplier Service
State management, enrichment, APIs
        |
        v
Cassandra / Vehicle Event State
        |
        v
Consumer Channels
myAudi app, vehicle, workshop, internal systems
```

---

# 7. Role of the Dispatcher Service

## 7.1 Business Purpose of Dispatcher

The Dispatcher is the **entry point for event data**.

It receives raw vehicle-related events from upstream systems and prepares them for STO.

A simple way to describe it:

> The Dispatcher converts messy external event data into clean internal vehicle events.

It does not own the long-term vehicle state. It is not the main customer API service. Its purpose is to understand, validate, enrich, and route incoming events.

---

## 7.2 Why Dispatcher Is Needed

Upstream systems are not consistent.

They may differ in:

- Event format
- Vehicle identifier type
- Region
- Brand coding
- Event naming
- Payload structure
- Source-system ownership
- Reset behavior
- Notification behavior

If every downstream system had to understand all these variations, the architecture would become messy and fragile.

The Dispatcher centralizes this responsibility.

It acts as a **translation and routing layer** between upstream systems and STO’s internal event model.

---

## 7.3 Main Business Responsibilities of Dispatcher

## 1. Receive Events from Upstream Systems

Dispatcher receives raw service, warning, recall, predictive, and quality-related events.

Examples of upstream event families:

- ACDC events
- Predictive maintenance events
- Quality/customer information events
- Regional E3 events
- Reset-related events
- Notification-related internal events

Business meaning:

> Dispatcher is the point where external vehicle signals enter STO.

---

## 2. Validate Incoming Events

Before an event can affect a customer or be stored for a vehicle, STO needs confidence that the event is meaningful.

Dispatcher checks whether the event has enough information to proceed.

For example:

- Is there a vehicle identifier?
- Is the brand known?
- Is the event type valid?
- Is the event payload understandable?
- Is the source event structurally usable?

Business meaning:

> STO should not expose invalid, incomplete, or corrupted events to customers or workshops.

---

## 3. Normalize the Event

Different systems may describe similar things differently.

Dispatcher normalizes these differences into a common internal representation.

For example:

- Brand codes are mapped to business brand values.
- Event fields are extracted into expected internal fields.
- Source-specific formats are converted into STO’s internal event language.
- Event metadata is prepared for downstream use.

Business meaning:

> After Dispatcher processing, the rest of STO does not need to know every upstream system’s unique format.

---

## 4. Resolve Vehicle Identity

This is one of the most important business functions.

STO is VIN-centric, but upstream systems may not always provide the actual VIN. Sometimes they provide a pseudonymous vehicle identifier or another technical identifier.

Dispatcher resolves this to the correct vehicle identity and context.

Business meaning:

> STO cannot show information to the right customer or workshop unless it knows exactly which vehicle the event belongs to.

Vehicle context may include:

- VIN
- Brand
- Country
- Region
- Market
- Home backend region

This context is essential because communication rules may depend on region, market, and brand.

---

## 5. Apply Routing and Event Configuration

Not every event should be treated the same way.

Dispatcher uses configuration to decide how an event should be handled.

Configuration may answer questions like:

- Should this event be processed?
- Which downstream topic or flow should receive it?
- Is it relevant for the frontend?
- Is it relevant for the vehicle?
- Can it trigger a notification?
- Can it be reset later?
- Which source system owns the reset path?
- Which market/brand rules apply?

Business meaning:

> Event behavior is policy-driven, not hardcoded. Audi can adapt business rules by configuration.

---

## 6. Publish Clean Internal Events

Once the event is validated, normalized, and linked to a vehicle, Dispatcher publishes it into the internal STO flow.

At this point, the event is no longer a raw upstream message. It is a clean STO event ready for persistence and serving.

Business meaning:

> Dispatcher turns external signals into reliable business events.

---

## 7. Handle Bad or Unprocessable Events

If an event cannot be processed, STO should not simply lose it or crash.

Dispatcher separates problematic events from the main flow.

Examples:

- Invalid payload
- Missing vehicle identifier
- Failed vehicle identity resolution
- Unsupported format

Business meaning:

> Bad events are isolated and recoverable. They do not block valid vehicle events from being processed.

---

## 8. Forward Reset Requests

Reset is an important user journey.

A customer or vehicle may request that a warning/event be reset. The Supplier receives the customer-facing request, but the original source system may need to know about the reset.

Dispatcher handles this because it understands upstream routing.

Business meaning:

> The customer-facing API remains simple, while Dispatcher hides the complexity of source-system reset routing.

---

## 9. Trigger Notifications

Dispatcher also participates in push notification delivery.

When Supplier determines that a meaningful customer-relevant event has been stored, it can trigger a notification flow. Dispatcher then evaluates notification policy and communicates with notification systems.

Business meaning:

> Notification delivery is controlled and policy-based, not directly coupled to raw event ingestion.

---

# 8. Role of the Supplier Service

## 8.1 Business Purpose of Supplier

Supplier is the **system of record and API provider** for STO.

It owns the current service-event state for each vehicle and serves that state to consumers.

Simple description:

> The Supplier stores what STO currently knows about a vehicle and answers requests from apps, vehicles, workshops, and internal systems.

---

## 8.2 Why Supplier Is Needed

Dispatcher prepares events, but the business needs a reliable place to answer questions like:

- What active events exist for this VIN?
- Which events should the customer see?
- Which events should the vehicle display?
- Which events are relevant for a workshop?
- What localized text should be shown?
- What is the current recall or campaign status?
- Is this event still active or already resolved?
- Can this event be reset?
- Should this event be deleted for GDPR reasons?

Supplier is responsible for those answers.

---

## 8.3 Main Business Responsibilities of Supplier

## 1. Consume Clean Internal Events

Supplier receives normalized STO events from Dispatcher.

These events are already prepared and vehicle-linked.

Business meaning:

> Supplier does not need to understand every upstream source system. It focuses on business state.

---

## 2. Maintain Current Vehicle Event State

Supplier stores the current state of events for each vehicle.

It does not primarily act as a historical archive. Its main responsibility is the current truth.

For example:

For VIN `WAU123`, Supplier may know:

- Recall campaign `R001` is open.
- Warning `W042` is active.
- Predictive maintenance event `P100` is passive.
- Quality information event `Q123` is visible to the frontend.

Business meaning:

> Supplier provides the latest relevant service picture of a vehicle.

---

## 3. Prevent Incorrect State Updates

Vehicle event streams can contain duplicates or older messages.

Supplier protects the business state by ensuring older information does not overwrite newer information.

Business example:

If STO already knows that a recall is currently open, an old duplicate event should not incorrectly mark it as resolved.

Business meaning:

> Supplier protects customers and workshops from seeing stale or incorrect service information.

---

## 4. Apply Event Lifecycle Rules

Events do not live forever.

Some are active for a long time. Some disappear quickly after they become passive. Some are resolved but remain visible briefly. Some must be deleted for legal reasons.

Supplier applies lifecycle rules such as:

- Active event visibility
- Passive event expiry
- Resolved event retention
- GDPR deletion
- Configuration-based retention behavior

Business meaning:

> STO keeps information current and avoids showing outdated or irrelevant service events.

---

## 5. Serve Customer-Facing APIs

Supplier exposes APIs used by the myAudi app and portal.

These APIs return customer-facing vehicle service information.

The response must be:

- Specific to the vehicle
- Relevant to the customer
- Filtered by channel
- Localized
- Easy to display
- Not overloaded with internal technical data

Business meaning:

> Supplier is the backend source for what the customer sees in the digital Audi experience.

---

## 6. Serve In-Vehicle APIs

Supplier also supports in-vehicle communication.

Vehicles may request current service or warning information and display it in the car.

Business meaning:

> STO allows relevant service information to reach the driver directly inside the vehicle.

---

## 7. Serve Workshop and Partner APIs

Supplier provides data for service partners and workshops.

This supports:

- Service appointment preparation
- Recall awareness
- Campaign completion
- Workshop planning
- Service advisor support

Business meaning:

> Workshops can better understand a vehicle’s service needs before or during customer interaction.

---

## 8. Enrich Responses with External Business Data

Supplier does not only return stored event rows.

It may enrich responses using additional business information, such as:

- Localized CMS/zFDI text
- Recall campaign details
- Online update status
- Quality/customer information
- Region-specific response mapping

Business meaning:

> Supplier turns raw service state into meaningful, customer-readable, action-oriented information.

---

## 9. Support Reset Requests

Supplier exposes the customer-facing reset API.

When a user resets an event, Supplier:

- Authenticates the request
- Checks whether the event exists
- Checks whether reset is allowed
- Starts an asynchronous reset flow

Business meaning:

> The customer can initiate reset actions without needing to know anything about backend source systems.

---

## 10. Support Configuration Management

Supplier includes configuration-related functionality.

Configuration may define:

- Default event behavior
- Visibility rules
- Channel behavior
- Event mapping
- Reset rules
- Response filtering
- Initial setup values

Business meaning:

> Business rules can evolve without major code changes.

---

## 11. Support GDPR/DSGVO Workflows

Supplier provides privacy-related functionality.

This includes:

- Inquiry into stored vehicle-related data
- Deletion of data for a VIN
- Deletion by event type or event ID
- Scheduled/legal cleanup processes

Business meaning:

> STO supports data protection compliance and the right to be forgotten.

---

# 9. Business Flow: From Raw Event to Customer Value

This is the complete business journey.

## Step 1: Upstream System Detects or Publishes an Event

An upstream system identifies a relevant vehicle-related event.

Examples:

- A recall campaign applies to a vehicle.
- A predictive maintenance signal is generated.
- A quality information event is created.
- A warning state is received.
- A service campaign becomes active.

The upstream system publishes this information into the STO ecosystem.

---

## Step 2: Dispatcher Receives the Event

Dispatcher receives the raw event.

At this point, the event may still be technical, incomplete, source-specific, or not directly usable by customer-facing systems.

Dispatcher starts preparing it.

---

## Step 3: Dispatcher Validates and Normalizes the Event

Dispatcher checks whether the event is valid and converts it into STO’s internal format.

It makes sure the event can be understood consistently downstream.

---

## Step 4: Dispatcher Identifies the Vehicle

Dispatcher resolves the vehicle identity.

This is critical because STO is built around VIN-specific communication.

Without correct vehicle identity, the event cannot be safely shown to a customer, workshop, or vehicle.

---

## Step 5: Dispatcher Applies Business Routing Rules

Dispatcher checks the event configuration.

It determines how the event should move forward.

For example:

- Is this event relevant for STO?
- Should it be forwarded to Supplier?
- Could it trigger notification later?
- Which brand/region rules apply?
- Is it resettable?
- Which channel flags apply?

---

## Step 6: Dispatcher Publishes a Clean Internal Event

Dispatcher sends a clean event into STO’s internal stream.

Now the event is ready for state management.

---

## Step 7: Supplier Consumes the Event

Supplier receives the normalized event and checks the current known state for that vehicle.

It determines whether this event represents:

- A new event
- An update to an existing event
- A duplicate event
- An older stale event
- A passive/resolved event

---

## Step 8: Supplier Updates Vehicle State

If the event is meaningful, Supplier updates the current vehicle state.

This becomes part of the authoritative STO view for that VIN.

---

## Step 9: Notification May Be Triggered

If the event is customer-relevant and notification-eligible, a notification flow can be triggered.

Example:

> A high-criticality active event may result in a push notification to the customer.

---

## Step 10: Consumer Requests Vehicle Information

A customer, vehicle, workshop, or partner system requests service information for a VIN.

Supplier receives the request.

---

## Step 11: Supplier Builds the Response

Supplier retrieves the current state and applies business rules.

It may:

- Filter by channel
- Filter by visibility
- Add localized text
- Add recall campaign details
- Add ORU/update status
- Deduplicate overlapping information
- Prioritize events
- Format the response for the consumer

---

## Step 12: Customer or Workshop Receives Actionable Information

The final result is a meaningful business outcome.

Examples:

- Customer sees an open recall in myAudi.
- Vehicle displays a relevant warning.
- Workshop sees open campaigns before service.
- Compliance system deletes data on request.
- Internal tools manage event configuration.

---

# 10. Business Use Cases

## Use Case 1: Customer Sees an Open Recall in myAudi

### Scenario

Audi identifies that a specific VIN is affected by a recall campaign.

### Flow

1. Recall/campaign system publishes an event.
2. Dispatcher receives and normalizes it.
3. Dispatcher identifies the VIN.
4. Supplier stores the event as current vehicle state.
5. Customer opens myAudi app.
6. Supplier returns the recall as part of the vehicle’s active service information.
7. Customer sees the recall and can take action.

### Business Value

- Faster recall awareness
- Higher recall completion rate
- Improved customer safety
- Better digital customer experience

---

## Use Case 2: Predictive Maintenance Warning

### Scenario

A vehicle generates a predictive maintenance event.

### Flow

1. Predictive system sends an event.
2. Dispatcher prepares and routes it.
3. Supplier stores it.
4. If configured, a notification is triggered.
5. Customer sees the maintenance warning in the app or vehicle.

### Business Value

- Early issue detection
- Reduced breakdown risk
- More proactive service scheduling
- Increased customer satisfaction

---

## Use Case 3: Workshop Prepares for a Customer Visit

### Scenario

A customer books a workshop appointment.

### Flow

1. Workshop system queries STO by VIN.
2. Supplier returns current active events and campaigns.
3. Workshop sees open recall/service information.
4. Advisor prepares parts, time, and guidance.

### Business Value

- Better workshop efficiency
- Shorter diagnosis time
- Fewer missed campaigns
- Better customer trust

---

## Use Case 4: Customer Resets a Warning

### Scenario

A customer dismisses or resets a resettable event in the app.

### Flow

1. Customer calls reset action through app.
2. Supplier validates that reset is allowed.
3. Supplier starts reset request.
4. Dispatcher routes reset to correct upstream source system.
5. Source system later sends updated event state.
6. Supplier updates the vehicle state.

### Business Value

- Better customer control
- Clean separation between frontend and backend systems
- Consistent state update through normal event flow

---

## Use Case 5: Localized Customer Message

### Scenario

A customer in Germany opens the app, and the event should be shown in German.

### Flow

1. Supplier receives app request with language/country context.
2. Supplier retrieves the event state.
3. Supplier enriches event using localized text data.
4. Customer receives German text.

### Business Value

- Market-appropriate communication
- Better customer understanding
- Reusable global event model with local content

---

## Use Case 6: GDPR Data Deletion

### Scenario

A privacy request requires deletion of stored vehicle-related data.

### Flow

1. Compliance system calls DSGVO/GDPR endpoint.
2. Supplier identifies stored data for the VIN or event.
3. Supplier deletes the relevant state.
4. Inquiry/deletion response is returned.

### Business Value

- Regulatory compliance
- Controlled deletion process
- Trustworthy data governance

---

# 11. Channel-Specific Business Behavior

One of STO’s most important business features is that it does not expose the same data to every channel.

## 11.1 Frontend Visibility

Some events are suitable for customer app display.

These should be:

- Understandable
- Actionable
- Relevant
- Localized
- Not overly technical

Example:

> “Service campaign available. Please contact your Audi partner.”

---

## 11.2 Vehicle Visibility

Some events are suitable for in-vehicle display.

These may be:

- Time-sensitive
- Operationally relevant while driving
- Connected to vehicle status
- Suitable for dashboard or infotainment presentation

---

## 11.3 Workshop Visibility

Workshops may need more operational or complete information.

They may see events that are not necessarily intended for direct customer communication.

---

## 11.4 Notification Eligibility

Not every event should trigger a push notification.

Notification may depend on:

- Event type
- Criticality
- Brand
- Market
- Customer channel
- Configuration
- Whether the event is new or already known

This avoids notification fatigue and ensures only meaningful events are pushed.

---

# 12. High-Level Data Ownership

At a business level, ownership looks like this:

| Area | Owner |
|---|---|
| Raw upstream event formats | Source systems |
| Event normalization and routing | Dispatcher |
| Current vehicle event state | Supplier |
| Customer-facing service event response | Supplier |
| Vehicle-facing response | Supplier |
| Workshop-facing response | Supplier |
| Notification routing | Dispatcher |
| Reset API entry point | Supplier |
| Reset routing to source system | Dispatcher |
| Configuration behavior | Shared, consumed by both |
| GDPR deletion | Supplier |

---

# 13. Why the Dispatcher/Supplier Split Is Valuable

This is one of the strongest architecture points.

## 13.1 Separation of Concerns

Dispatcher focuses on:

> “Can I understand this event and route it correctly?”

Supplier focuses on:

> “What is the current truth for this vehicle, and how should I serve it?”

This makes the system easier to maintain and reason about.

---

## 13.2 Independent Scaling

Event ingestion and API serving have different load patterns.

There may be:

- Event bursts from upstream systems
- High app traffic from customers
- Workshop traffic during business hours
- Batch-like campaign updates
- Regional spikes

By separating the services, Audi can scale the ingestion side and the serving side independently.

---

## 13.3 Better Failure Isolation

If an upstream source sends bad events, Dispatcher can isolate those issues.

If an external enrichment service is slow, Supplier can still serve available data.

If the API layer is under load, Dispatcher can continue receiving events.

This reduces the chance that one problem brings down the entire platform.

---

## 13.4 Clear Business Ownership

The split creates clear responsibility:

- Dispatcher owns the input complexity.
- Supplier owns the customer-facing truth.

This is easier to explain, operate, and evolve.

---

# 14. Configuration-Driven Business Rules

STO is designed to support changing business rules.

Instead of hardcoding every event behavior, configuration can define how events behave.

Examples of configurable behavior:

- Should an event be shown in the app?
- Should it be shown in the vehicle?
- Should it trigger notification?
- Is it resettable?
- What priority does it have?
- How long should it remain active?
- Which market or brand does it apply to?
- How should recall types be mapped?
- Which event patterns match which business rules?

Business value:

> Audi can adapt communication behavior for different markets, brands, vehicle generations, and event types without redesigning the whole system.

---

# 15. Resilience from a Business Perspective

STO is designed so that temporary failures do not immediately destroy the customer experience.

## 15.1 Bad Events Do Not Stop the System

If one incoming event is invalid, it should not stop the processing of other vehicles.

Business value:

> One bad source message should not impact customers whose events are valid.

---

## 15.2 Partial Responses Are Better Than Total Failure

If Supplier cannot enrich recall data from an external source, it may still return available STO events.

Business value:

> The app can still show useful information instead of showing nothing.

---

## 15.3 Current State Is Protected

Supplier avoids letting older or duplicate information overwrite newer vehicle state.

Business value:

> Customers and workshops see the most accurate available vehicle status.

---

## 15.4 Privacy Workflows Are Built In

GDPR/DSGVO support is part of the platform, not an afterthought.

Business value:

> Legal compliance is operationally supported.

---

# 16. End-to-End Business Example

Let’s walk through a realistic example.

## Scenario

A vehicle is affected by a service campaign. Audi wants the customer to know through the myAudi app and wants workshops to see it during service planning.

## End-to-End Flow

1. An upstream campaign/event system publishes the event.
2. Dispatcher receives it.
3. Dispatcher validates that the event has required information.
4. Dispatcher identifies the correct vehicle.
5. Dispatcher normalizes the event into STO’s internal format.
6. Dispatcher applies configuration and routes it forward.
7. Supplier receives the clean event.
8. Supplier checks whether the vehicle already has this event.
9. Supplier updates the current state for that VIN.
10. If eligible, a notification is triggered.
11. Customer opens myAudi app.
12. Supplier returns the event with customer-friendly, localized text.
13. Workshop later queries the same VIN.
14. Supplier returns the campaign information for service planning.
15. If the event is resolved later, an update flows through the same pipeline.
16. Supplier updates the current state so the customer and workshop no longer see outdated information.

## Business Result

- Customer is informed early.
- Workshop is prepared.
- Audi improves recall/service communication.
- Vehicle data remains consistent across channels.

---

# 17. What the Application Ultimately Does

In business terms, the application does five big things:

## 1. Ingests Vehicle Service Signals

It receives service, recall, warning, predictive, and quality signals from multiple upstream systems.

## 2. Converts Them into VIN-Specific Events

It identifies the correct vehicle and normalizes the event into a common business language.

## 3. Maintains Current Vehicle Service State

It stores what is currently true for each VIN.

## 4. Serves Different Consumers Through Different Views

It provides tailored views for:

- Customer app
- Vehicle display
- Workshop systems
- Partner integrations
- Internal tools
- Compliance systems

## 5. Ensures Business Reliability and Compliance

It supports:

- Error isolation
- Duplicate/stale event protection
- Configurable business rules
- Privacy deletion
- Market/channel-specific behavior
- Graceful degradation

---

# 18. Interview-Ready Business Explanation

You can say:

> Audi STO is a VIN-centric service communication platform. Its purpose is to bring vehicle service, recall, warning, and predictive maintenance information to customers and workshops much earlier than traditional workshop-based or letter-based communication.
>
> The system receives raw events from many upstream systems. The Dispatcher service acts as the ingestion and routing layer. It validates events, identifies the vehicle, normalizes the data, applies business routing rules, and sends clean events into the internal platform.
>
> The Supplier service is the state and API layer. It stores the current event state for each VIN and exposes that information to the myAudi app, in-vehicle systems, workshop portals, internal systems, reset flows, configuration tools, and GDPR processes.
>
> The business value is that Audi can provide customers with timely, personalized, localized service information, help workshops prepare better, improve recall completion, reduce delayed communication, and support compliance requirements. The architecture separates raw event processing from customer-facing state management, which makes the platform scalable, resilient, and easier to evolve.

---

# 19. Short Version for Interviews

If you need a 60-second version:

> STO, or Service Technik Online, is Audi’s backend platform for digital vehicle service communication. It processes recall, warning, predictive maintenance, and service campaign events for individual vehicles and makes them available to customers, vehicles, workshops, and internal systems.
>
> The two main services are Dispatcher and Supplier. Dispatcher is the ingestion service. It receives raw events from upstream systems, validates and normalizes them, identifies the correct VIN, applies routing rules, and forwards clean events. Supplier is the state and serving service. It stores the current vehicle event state and exposes APIs for the myAudi app, in-vehicle systems, service partners, reset actions, configuration, and GDPR deletion.
>
> The main business value is proactive communication. Customers no longer need to wait for workshop visits or letters to know about important service events. Workshops get better preparation data, and Audi gets a scalable, configurable, and compliant platform for VIN-specific service communication.

---

# 20. One-Line Memory Hook

> **Dispatcher turns raw vehicle signals into clean STO events. Supplier turns clean STO events into reliable customer, vehicle, workshop, and compliance experiences.**