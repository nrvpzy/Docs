# Detailed Business + Architecture Report: JLR MCP Toolbox Platform

This project is best understood as an **internal enterprise platform for managing the full lifecycle of MCP vehicle orders**, from employee eligibility and vehicle configuration through order creation, allocation, Vista/SAP operations, handover, return, finance reporting, and admin access control.

At a high level:

> JLR had multiple operational teams managing employee vehicle orders through emails, spreadsheets, SAP/Vista portals, manual file uploads, and disconnected systems. This application centralized those workflows into one secure Toolbox platform with SSO, RBAC, order management, vehicle master data, finance reporting, employee data, dashboards, and operational reports.

---

# 1. One-Minute Interview Summary

If an interviewer asks, **“What was this project about?”**, you can say:

> This was an internal JLR Toolbox platform built to digitize and centralize the MCP vehicle ordering process. Previously, a lot of the workflow was handled through email chains, spreadsheets, SAP/Vista portals, and manual coordination between operations, finance, HR, and vehicle teams. The application brought those workflows into a microservices-based platform where users could securely log in through enterprise SSO, create and manage vehicle orders, allocate vehicles, manage spare vehicles, process Vista and SAP updates, upload bulk files, generate reports, manage employee data, and control access using role-based permissions.
>
> The architecture had a Next.js/React frontend and several Spring Boot microservices: admin/RBAC, ADFS bridge for SSO, vehicle order service, vehicle data service, employee service, finance service, and a local gateway/reverse proxy. The core value was reducing manual handoffs and giving business teams one controlled place to manage the full order lifecycle.

---

# 2. What Business Problem Was JLR Solving?

Before this platform, the process was fragmented.

Different teams likely used:

- Email chains
- Excel spreadsheets
- Manual approvals
- SAP portal interactions
- Vista data updates
- Finance reports generated separately
- HR employee extracts
- Pricing matrix files
- Manual order tracking
- Separate communication between departments

This created problems:

## 2.1 Manual handoffs

A vehicle order might require input from:

- Employee/fleet admin
- HR/employee-data team
- Vehicle operations
- Allocation team
- Vista team
- SAP team
- Finance team
- Admin/security team

If these handoffs happen over email, delays are common.

---

## 2.2 No single source of truth

Different teams may maintain different spreadsheets or reports.

For example:

- One team tracks employee eligibility.
- Another tracks vehicle orders.
- Another tracks Vista status.
- Another tracks SAP handover.
- Another tracks finance/loan data.

This leads to mismatched data.

---

## 2.3 Poor visibility

Managers and operations users need to answer questions like:

- How many orders are awaiting allocation?
- Which orders are delayed?
- Which vehicles are ready for handover?
- Which orders need SAP handover?
- Which vehicles are due for return?
- Which spare vehicles are available?
- Which employee has an active order?
- Which orders are missing VIN/common order numbers?
- Which orders are excluded from Vista/SAP reports?

Without a centralized system, these questions take time.

---

## 2.4 Risk of duplicate or invalid orders

If order creation is handled manually, users can accidentally:

- create duplicate active orders for the same employee
- assign unavailable vehicles
- miss required validations
- enter inconsistent employee data
- use outdated pricing/vehicle configuration

---

## 2.5 Inconsistent access control

Before centralized RBAC, access may be managed manually or inconsistently.

Some users might see or perform actions they should not.

For example:

- Finance users should not necessarily manage roles.
- Operations users should not necessarily upload pricing matrix data.
- Admin users may need access to users/roles.
- SAP report users may only need SAP-related screens.

---

# 3. What the Application Does

The platform provides a secure, centralized system for:

1. Enterprise login using JLR SSO/ADFS.
2. Role-based access control.
3. Vehicle order creation and lifecycle management.
4. Employee data lookup and HR file processing.
5. Vehicle master data and pricing matrix management.
6. Allocation of vehicles/orders.
7. Vista bulk order processing and error handling.
8. SAP handover/status/return reports.
9. Spare vehicle/order management.
10. Finance reports such as loan agreements and annual statements.
11. Dashboards for order and spare vehicle visibility.
12. File uploads and Excel report generation.

---

# 4. The Main Business Domains

The application can be understood through seven business domains.

```text
1. Identity and Access
2. Employee Data
3. Vehicle Master Data
4. Vehicle Order Lifecycle
5. Operational Integrations: Vista, SAP, Manheim
6. Finance and Reporting
7. Frontend Workflow / Dashboards
```

---

# 5. Domain 1: Identity and Access

Owned mainly by:

- `toolboxadfsbridge`
- `toolboxadmin`
- `toolboxui`
- security config in all microservices

## Purpose

Only authenticated and authorized users should access the system.

The platform uses:

- ADFS/OIDC SSO for authentication
- JWT-based microservice security
- RBAC through users, roles, authorization codes

## Business value

This ensures:

- users log in with enterprise credentials
- access follows JLR corporate identity policies
- users only see the pages/actions they are allowed to use
- sensitive operations like cancel order, return vehicle, role management, pricing uploads, and report generation are controlled

---

# 6. Domain 2: Employee Data

Owned by:

- `employee-data-service`

Important capabilities:

- Create employee
- Update employee
- Delete employee
- Get employee by CDS ID
- Get employee by first name and last name
- Search employees
- Upload HR daily/monthly/leavers files
- Manage MCP vehicle schemes

## Why employee data matters

Vehicle orders are tied to employees.

An order needs employee information like:

- CDS ID
- first name
- last name
- phone
- email
- payroll number
- pay grade
- address
- post code
- eligibility
- scheme
- tag number

The employee service provides a controlled employee source for order workflows.

---

## HR file processing

The employee service processes files like:

- HR daily file
- HR monthly file
- HR leavers file

This allows the system to keep employee records updated without manual entry.

Flow:

```text
HR file uploaded
   ↓
Employee service parses file
   ↓
Rows mapped into Employee data
   ↓
Existing employees updated / new employees created
   ↓
Errors collected and returned
```

## Business value

- Reduces manual employee entry
- Keeps employee data aligned with HR data
- Helps validate whether a person is eligible for an order
- Reduces duplicate or inconsistent user records

---

# 7. Domain 3: Vehicle Master Data

Owned by:

- `mcp-vehicle-data-service`

This service owns the data required to configure and validate vehicles.

Important data:

- Brands
- Models
- Model years
- Derivatives
- Derivative finance
- Exterior colours
- Interior colours
- Chargeable options
- Dealer-fit options
- Must-fit options
- Pay-grade-to-derivative mappings
- Pricing matrix
- Vehicle data
- Vehicle mileage history
- Vehicle registration history
- Vehicle status history
- Spare vehicle data
- Reserved/spare history

---

## Pricing matrix

The pricing matrix is a key business input.

It defines what vehicles/options are available and their financial values.

The service can process a pricing matrix Excel file and save:

- brand details
- model details
- model year details
- derivative details
- derivative finance
- exterior colours
- interior colours
- chargeable options
- must-fit options

Flow:

```text
Pricing matrix uploaded
   ↓
Vehicle data service parses Excel
   ↓
Extracts vehicle configuration and finance data
   ↓
Saves reference data into database
   ↓
Frontend/order service uses this data during order creation
```

---

## Why vehicle data service is separate

Vehicle master data changes differently from order data.

Orders are transactional.

Vehicle master data is reference/configuration data.

Separation gives:

- clear ownership
- reusable vehicle data across order, spare, pricing, and reports
- less coupling inside the order service
- easier pricing matrix maintenance

---

# 8. Domain 4: Vehicle Order Lifecycle

Owned by:

- `mcp-vehicle-order`

This is the heart of the system.

It manages:

- order creation
- order modification
- order cancellation
- vehicle return
- order history
- reorder history
- allocation
- spare orders
- bulk uploads
- Vista workflows
- SAP workflows
- late orders
- dashboards and counts
- reports/export data

---

# 9. What Is a Vehicle Order?

A vehicle order is not one simple record.

It is a combination of multiple business details:

## Vehicle details

- brand
- model
- model year
- derivative
- exterior colour
- interior colour
- CI code
- VIN
- common order number
- ECCO order number
- order status
- required by date
- build period
- handover dates
- return dates

## Employee/user details

- CDS ID
- name
- email
- phone
- payroll number
- pay grade
- tag number
- address
- post code
- pickup/drop location

## Finance details

- loan value
- monthly deduction
- duration
- agreement value
- exception template

## Options/accessories

- chargeable options
- must-fit options
- dealer-fitted accessories

## Workflow data

- accepted by sales
- call over
- dealer delivered
- PDI status
- invoice date
- booked handover
- actual handover

## History

- created
- updated
- cancelled
- returned
- allocation changes
- Vista/SAP-related events

---

# 10. Vehicle Order Lifecycle: End-to-End

The lifecycle can be explained like this:

```text
Employee selected / order requested
   ↓
Vehicle configured using master data
   ↓
Order submitted
   ↓
Employee/order/vehicle validations
   ↓
Initial status calculated
   ↓
Order saved
   ↓
Awaiting allocation / submit to build / other status
   ↓
Allocation team assigns or progresses order
   ↓
Vista updates common order number / errors
   ↓
SAP reports handle handover/return/status
   ↓
Vehicle handed over
   ↓
Vehicle later returned
   ↓
Mileage/registration/history updated
   ↓
Order closed or moved to returned/off-lease states
```

---

# 11. Order Creation

Main API:

- `VehicleOrderController.saveOrder`

Service:

- `VehicleOrderService.createOrder`

## What happens

```text
UI submits order form
   ↓
Order service receives request
   ↓
Validates employee details
   ↓
Checks duplicate active orders
   ↓
Checks vehicle availability
   ↓
Calculates initial status
   ↓
Saves order details
   ↓
Saves user details
   ↓
Saves finance details
   ↓
Saves options/accessories
   ↓
Creates history
   ↓
Returns created order
```

## Business value

This replaces manual creation and validation.

Instead of someone manually checking spreadsheets and emailing other teams, the system applies rules at submission time.

---

# 12. Order Modification

Main API:

- `VehicleOrderController.modifyOrder`

Service:

- `VehicleOrderService.updateOrder`

## What happens

```text
User edits order
   ↓
System fetches current order
   ↓
Checks if current status allows modification
   ↓
Checks if employee details changed
   ↓
Prevents duplicate order for changed employee
   ↓
Checks CI code changes
   ↓
Recalculates status
   ↓
Updates order data
   ↓
Creates history
```

## Business value

Changes are controlled, validated, and auditable.

---

# 13. Order Cancellation

Main API:

- `VehicleOrderController.cancelVehicleOrder`

Service:

- `VehicleOrderService.cancelVehicleOrder`

## What happens

```text
User requests cancellation
   ↓
Order is fetched
   ↓
Status changed to CANCELLED
   ↓
History event created
```

## Business value

Cancellation is no longer an email instruction; it becomes a controlled lifecycle action.

---

# 14. Vehicle Return

Main API:

- `VehicleOrderController.returnVehicle`

Service:

- `VehicleOrderService.returnVehicle`

The return request includes:

- return mileage
- return registration number
- vehicle order ID
- VIN/common order number context

## What happens

```text
User submits return vehicle request
   ↓
Order service updates return details
   ↓
Order service calls vehicle-data-service to create vehicle history
   ↓
Mileage/registration history is saved
   ↓
Order status updated
   ↓
Order history created
```

## Business value

This connects the order lifecycle to vehicle history. Instead of return details being manually passed around, they are recorded in the system.

---

# 15. Allocation Workflow

Main APIs:

- `McpOrderController.getOrdersForAllocation`
- `McpOrderController.getOrdersAgainstDerivativesForAllocation`
- `McpOrderController.saveOrderAllocation`

## What allocation means

Allocation is where operations teams decide which available vehicle/order should be assigned to which employee order.

The system provides:

- list of orders awaiting allocation
- filtering by derivative
- save allocation decisions

Flow:

```text
Operations user opens allocation screen
   ↓
System shows allocation-ready orders
   ↓
User filters by derivative/status/etc.
   ↓
User selects allocation
   ↓
System saves allocation decision
   ↓
Order status/workflow progresses
```

## Business value

This replaces manual matching through spreadsheets or emails.

---

# 16. Vista Workflows

Owned mainly by:

- `OrderForVistaController`
- `OrderForVistaService`

Important capabilities:

- get bulk orders for Vista
- get orders with status in build/awaiting CON
- update Vista CON from CSV
- update Vista errors from XLSX
- export Vista data
- exclude orders from Vista export

## What is Vista doing here?

Vista appears to be an external operational system used in vehicle ordering/build tracking.

The Toolbox system prepares and consumes data for Vista.

Examples:

- Export order data for Vista
- Import common order number updates
- Import Vista error files
- Exclude certain orders from Vista processing

Flow:

```text
Toolbox identifies Vista-eligible orders
   ↓
User exports data for Vista
   ↓
Vista processes/updates data
   ↓
User uploads Vista CON/error file
   ↓
Toolbox updates order records
   ↓
History/status updated
```

## Business value

This reduces manual reconciliation between Toolbox order data and Vista system outputs.

---

# 17. SAP Workflows

SAP-related controllers/services include:

- `SAPStatusController`
- `SAPHandoverController`
- `SAPValidationController`
- `SapReturnController`
- `SpareToFCLGController`

Capabilities:

- SAP status/CON report
- SAP handover list
- SAP handover report
- Exclude SAP handover orders
- SAP return eligibility
- Generate SAP return report
- SAP validation file processing
- Spare to FCLG report

## What SAP is doing here

SAP is likely used for downstream finance/logistics/vehicle lifecycle processing.

The application prepares SAP-related lists and reports so operational teams can process handovers, returns, statuses, and exclusions.

Flow example: SAP handover

```text
Order becomes eligible for handover
   ↓
Appears in SAP handover list
   ↓
User reviews/excludes if needed
   ↓
System generates SAP handover report
   ↓
Data consumed by SAP process
```

Flow example: SAP return

```text
Vehicle becomes eligible for return
   ↓
Appears in SAP return list
   ↓
User reviews/excludes if needed
   ↓
System generates return report
```

## Business value

This gives controlled report generation and reduces SAP portal/manual spreadsheet work.

---

# 18. Spare Vehicle / Spare Order Management

Two services are involved:

## `mcp-vehicle-order`

Manages spare orders and assignment:

- create spare order with details
- get spare orders
- assign spare order to user
- delete user details from spare order

## `mcp-vehicle-data-service`

Manages spare vehicle state:

- create spare
- find spare
- reserve spare
- unreserve spare
- update spare
- remove spare
- spare history
- reserved history

## Business meaning

A spare vehicle/order is likely used when:

- a user needs a replacement
- allocation requires temporary vehicles
- vehicles are held for operational reasons
- spare stock needs to be tracked

Flow:

```text
Spare vehicle/order created
   ↓
Spare appears in spare list
   ↓
User reserves or assigns spare
   ↓
System records reserved/spare history
   ↓
Spare can be unreserved, removed, or assigned
```

## Business value

Spare vehicles become visible and controlled, instead of tracked manually.

---

# 19. Finance Service

Owned by:

- `mcp-finance-service`

Capabilities:

- loan agreement report
- annual statement report
- CVMS report upload
- JLR net payment report upload
- JLR net payment report filtering
- lookup net payment by VIN
- vehicle loan history
- vehicle loan value history

---

## Loan Agreement Flow

```text
User requests loan agreement for order
   ↓
Finance service fetches order details from order service
   ↓
Extracts finance/order data
   ↓
Performs calculations
   ↓
Maps values into report template
   ↓
Generates report as byte[]
   ↓
UI downloads PDF/report
```

The repo has report templates:

- `loan_agreement_report.jrxml`
- `loan_agreement_report_above_60k.jrxml`

This suggests JasperReports.

---

## Annual Statement Flow

```text
User requests annual statement
   ↓
Finance service generates statement report
   ↓
Returns byte[] to UI
```

---

## CVMS / Net Payment Uploads

```text
Finance file uploaded
   ↓
Finance service parses Excel
   ↓
Stores finance report data
   ↓
Users can search/filter/report later
```

## Business value

Finance data and documents are generated consistently, using order data from the platform.

---

# 20. Admin / RBAC Service

Owned by:

- `toolboxadmin`

Capabilities:

- manage users
- manage roles
- manage authorizations
- map users to roles
- map roles to authorization codes
- load authorization codes from JSON
- resolve a user’s authorization list

## Why this matters

The application has many sensitive actions:

- Create order
- Modify order
- Cancel order
- Return vehicle
- Upload Vista files
- Generate SAP reports
- Manage users
- Manage roles
- Upload pricing matrix
- Access finance reports

Not everyone should access everything.

RBAC gives fine-grained control.

---

# 21. Frontend Application

Owned by:

- `toolboxui`

This is the user-facing application.

It includes pages for:

- Dashboard
- MCP order
- MCP order list
- Awaiting allocation
- Assigned order list
- User orders
- Spare order
- Spare list
- Vista bulk order
- Vista confirmation
- SAP handover report
- SAP return report
- SAP status report
- Symmetry report
- Loan agreement
- Annual statement
- Pricing matrix
- Employee management
- Scheme management
- Role management
- User management

---

## Key frontend patterns

### Reusable data grid

Used for large operational lists.

### Filters/sorting/pagination

Used to search orders, SAP reports, Vista reports, employee lists, pricing matrix data.

### Authorization-aware sidebar/buttons

Uses authorization codes to show/hide routes and actions.

### Redux form state

Used for complex multi-step order/spare forms.

### Dashboard charts

Shows order/spare metrics and summaries.

---

# 22. Gateway / Reverse Proxy

Owned by:

- `toolboxlocalgw`

It routes traffic and handles redirect rewriting.

Important because SSO flows involve redirects, and internal service URLs often need to be rewritten for browser-facing URLs.

Business value:

- provides unified access path
- supports local/environment routing
- prevents exposing internal hostnames in redirects

---

# 23. ADFS Bridge

Owned by:

- `toolboxadfsbridge`

Handles enterprise SSO.

Capabilities:

- redirect user to ADFS
- receive OIDC principal after login
- get OAuth2 authorized client
- extract claims
- return user details
- support default mapping/redirect behavior

Business value:

- users use JLR enterprise login
- no app-specific passwords
- centralized identity policies
- easier offboarding/onboarding

---

# 24. End-to-End Business Flow: Creating an Employee Vehicle Order

This is the most important story.

```text
1. User logs into Toolbox using JLR SSO.

2. UI gets user permissions from toolboxadmin.

3. User opens MCP Order screen if they have SAVE_ORDER permission.

4. UI fetches employee data from employee service.

5. UI fetches vehicle configuration from vehicle-data-service:
   brand, model, model year, derivative, colours, options, finance values.

6. User fills order form:
   employee details, vehicle details, finance details, options/accessories.

7. UI submits order to mcp-vehicle-order.

8. Order service validates:
   employee identity,
   duplicate active order,
   vehicle availability,
   required fields.

9. Order service calculates initial status.

10. Order service saves:
   vehicle order details,
   user details,
   finance details,
   selected options,
   dealer-fitted accessories,
   workflow activity.

11. Order history is created.

12. Order appears in relevant operational lists:
   order list,
   awaiting allocation,
   Vista/SAP flows depending on status.
```

---

# 25. End-to-End Business Flow: Allocation

```text
1. Operations user opens Awaiting Allocation page.

2. UI checks authorization:
   VIEW_ORDERS_FOR_ALLOCATION.

3. UI calls mcp-vehicle-order allocation API.

4. User filters orders by derivative/status/date.

5. User saves allocation decision.

6. Order service updates allocation-related data.

7. Status changes if required.

8. History is recorded.
```

Business value:

> Vehicle allocation becomes trackable, searchable, and auditable.

---

# 26. End-to-End Business Flow: Return Vehicle

```text
1. User opens User Orders / Order List.

2. User selects Return Vehicle action.

3. UI checks RETURN_VEHICLE.PUT permission.

4. User enters return mileage and registration.

5. UI calls mcp-vehicle-order return API.

6. Order service updates order return fields.

7. Order service calls vehicle-data-service to save mileage/registration history.

8. Order status and history are updated.
```

Business value:

> Vehicle return is captured properly and downstream history/reporting remains consistent.

---

# 27. End-to-End Business Flow: Pricing Matrix Update

```text
1. Authorized user uploads pricing matrix.

2. Vehicle-data-service parses file.

3. It updates vehicle master data:
   brands, models, derivatives, colours, options, finance.

4. Updated configuration becomes available in order forms.

5. Future orders use latest vehicle/pricing data.
```

Business value:

> Vehicle/pricing configuration is centrally controlled.

---

# 28. End-to-End Business Flow: Finance Report

```text
1. User requests loan agreement for an order.

2. Finance service fetches order details from order service.

3. Finance service extracts and calculates values.

4. Jasper report template is populated.

5. PDF/report is returned to UI.
```

Business value:

> Finance documents are generated from system data instead of manual templates.

---

# 29. Why Microservices Made Sense

The application was split by business capability.

## `toolboxadmin`

Owns RBAC and admin access.

## `toolboxadfsbridge`

Owns SSO integration.

## `mcp-vehicle-order`

Owns transactional order lifecycle.

## `mcp-vehicle-data-service`

Owns vehicle master/reference data.

## `employee-data-service`

Owns employee records and HR ingestion.

## `mcp-finance-service`

Owns finance and report generation.

## `toolboxui`

Owns user experience.

## `toolboxlocalgw`

Owns routing/reverse proxy support.

---

## Benefits of this split

1. Clear domain ownership.
2. Independent deployment.
3. Easier scaling.
4. Better maintainability.
5. Finance/reporting load separated from order transactions.
6. Vehicle master data separated from order lifecycle.
7. Security/RBAC centralized.
8. SSO isolated from business services.

---

# 30. Before vs After

## Before

```text
Email chains
Excel sheets
SAP portal interactions
Vista manual updates
Manual allocation tracking
Manual employee checks
Manual finance report preparation
No single operational dashboard
Inconsistent access control
```

## After

```text
Central Toolbox UI
Enterprise SSO
Role-based access
Vehicle order APIs
Employee lookup
Vehicle master-data APIs
Order status tracking
Allocation workflow
SAP/Vista report workflows
Finance report generation
Audit history
Dashboards and searchable grids
```

---

# 31. Business Value

The application provided:

## 31.1 Faster processing

Order creation, allocation, return, and reporting workflows became API/UI-driven instead of email-driven.

## 31.2 Better visibility

Dashboards and filterable grids allowed users to quickly see what was happening.

## 31.3 Better control

RBAC prevented users from accessing unauthorized screens/actions.

## 31.4 Better data quality

Validations reduced duplicate orders, invalid employee data, unavailable vehicle assignments, and inconsistent order states.

## 31.5 Better auditability

Order history, spare history, reserved history, vehicle status history, mileage history, and registration history provided traceability.

## 31.6 Better integration

Vista, SAP, finance, HR, and pricing matrix workflows were pulled into one platform.

---

# 32. How to Explain This Project in an Interview

Use this structured answer.

## Start with the business problem

> The business problem was that MCP vehicle ordering was spread across email chains, spreadsheets, SAP/Vista portals, and manual team handoffs. This made it difficult to track order status, enforce validations, manage access, and generate consistent reports.

## Then explain the solution

> We built a centralized Toolbox platform with a React/Next frontend and Spring Boot microservices. It allowed users to securely log in with enterprise SSO, manage vehicle orders end-to-end, maintain employee and vehicle data, process SAP/Vista workflows, generate finance reports, and control access through RBAC.

## Then explain your main ownership

> My main contributions were in security — implementing SSO and RBAC — and in the order service — building APIs for order creation, modification, allocation, return, validation, status transitions, and filtering.

## Then explain the impact

> This reduced manual handoffs, improved visibility, prevented unauthorized access, and made operations faster and more auditable.

---

# 33. Interviewer: “What exactly does the platform do?”

Answer:

> It manages the complete lifecycle of MCP employee vehicle orders. Users can create orders, configure vehicles using master data, validate employee and vehicle eligibility, allocate orders, process Vista and SAP updates, manage spare vehicles, return vehicles, generate finance reports, and view dashboards. It also manages users, roles, and authorization so every team only accesses the functions they need.

---

# 34. Interviewer: “Who are the users of this system?”

Possible users:

- Fleet operations team
- Vehicle allocation team
- SAP processing users
- Vista processing users
- Finance/reporting team
- HR/employee-data administrators
- Pricing matrix/vehicle-data admins
- System admins managing users/roles
- Managers viewing dashboards

---

# 35. Interviewer: “What was the most complex part?”

Strong answer:

> The most complex part was the vehicle order lifecycle because an order can move through many states and can be affected by manual updates, allocation, Vista files, SAP reports, sales reports, cancellations, returns, and spare vehicle workflows. We handled this by centralizing status calculation and separating validation rules into dedicated classes.

---

# 36. Interviewer: “How did the application integrate with SAP and Vista?”

Answer:

> The application did not necessarily replace SAP or Vista. Instead, it centralized the operational workflow around them. It generated exports/lists for Vista and SAP, allowed users to upload Vista/SAP result files, updated order records based on those files, supported exclusions, and maintained history. This reduced manual reconciliation between internal order data and external systems.

---

# 37. Interviewer: “What was the role of finance service?”

Answer:

> Finance service handled finance-specific workflows such as loan agreement generation, annual statements, CVMS uploads, JLR net payments reports, and loan history. It could fetch order details from the order service, perform calculations, map values into Jasper report templates, and return downloadable reports.

---

# 38. Interviewer: “What was the role of vehicle-data-service?”

Answer:

> Vehicle-data-service owned master/reference data for vehicle configuration: brands, models, model years, derivatives, colours, options, dealer-fit options, must-fit options, finance values, pricing matrix, spare vehicle state, and vehicle history. The order service and UI depended on this data during order creation and vehicle operations.

---

# 39. Interviewer: “What was the role of employee service?”

Answer:

> Employee service managed employee master data and HR file ingestion. It allowed order workflows to look up employees by CDS ID or name, validate employee details, maintain HR-driven updates, and manage MCP vehicle schemes.

---

# 40. Interviewer: “What was the role of admin service?”

Answer:

> Admin service was the central RBAC service. It managed users, roles, and authorization codes. Authorization codes were seeded from a JSON catalog and assigned to roles. After SSO login, the UI fetched the user’s authorization codes and used them to control route and button visibility.

---

# 41. Interviewer: “How did this reduce manual work?”

Answer:

> Earlier, teams had to coordinate through emails and spreadsheets for order creation, allocation, status updates, SAP/Vista processing, and finance reports. The platform automated validations, centralized order state, exposed allocation-ready lists, generated reports, processed uploads, and maintained history. This reduced manual follow-ups and gave teams one source of truth.

---

# 42. Key Business Terms You Should Know

## MCP order

An employee vehicle order managed under the MCP process.

## CDS ID

Enterprise/user identifier used for employee lookup.

## Common Order Number / CON

A vehicle/order identifier used in downstream systems such as Vista/SAP workflows.

## VIN

Vehicle Identification Number.

## Derivative

Specific vehicle variant/configuration.

## CI code

Vehicle/order configuration-related code used in ordering/allocation.

## Vista

External/operational vehicle order/build system.

## SAP

Enterprise system used for downstream status, handover, return, finance/logistics processes.

## Spare vehicle

A vehicle/order available as spare/reserve/temporary allocation.

## Pricing matrix

Excel-based master data defining vehicle configurations, finance values, options, colours, etc.

---

# 43. Most Important Architecture Diagram in Words

Remember this:

```text
toolboxui
   |
   |-- login through ADFS bridge
   |-- gets permissions from admin service
   |-- calls business APIs
   v

toolboxlocalgw
   |
   |-- routes browser/API traffic
   |-- rewrites SSO redirect locations
   v

toolboxadfsbridge
   |
   |-- talks to ADFS
   |-- handles SSO login
   v

toolboxadmin
   |
   |-- users, roles, authorization codes
   v

mcp-vehicle-order
   |
   |-- order lifecycle
   |-- allocation
   |-- Vista/SAP order workflows
   v

mcp-vehicle-data-service
   |
   |-- vehicle master data
   |-- pricing matrix
   |-- spare/vehicle history
   v

employee-data-service
   |
   |-- employee records
   |-- HR uploads
   |-- schemes
   v

mcp-finance-service
   |
   |-- loan agreements
   |-- annual statements
   |-- finance reports
```

---

# 44. Best Final Project Pitch

Use this in interviews:

> This project was an internal JLR enterprise platform for managing MCP vehicle orders. The business process originally involved email chains, spreadsheets, SAP/Vista portal interactions, and manual coordination between operations, finance, HR, and vehicle teams. We built a centralized Toolbox application that digitized the workflow.
>
> Users logged in using enterprise SSO through ADFS. Access was controlled through RBAC in the admin service. The order service handled the core vehicle order lifecycle: create, update, allocate, cancel, return, validate, track history, and support Vista/SAP workflows. Vehicle-data-service owned pricing matrix and vehicle master data. Employee service owned employee records and HR file ingestion. Finance service generated loan agreements, annual statements, and finance reports. The React/Next UI tied these together through dashboards, forms, grids, filters, reports, uploads, and role-aware navigation.
>
> The main value was replacing manual handoffs with a secure, auditable, API-driven workflow that gave teams a single source of truth for vehicle orders and operational status.
