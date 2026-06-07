Absolutely — based on the repository map, this looks like a **microservices-based internal enterprise application** for managing JLR MCP vehicle orders, employees, vehicle/pricing data, finance/reporting, authorization, SSO, and UI workflows.

I’ll give you a strong architectural understanding first, then we can go deeper service by service or flow by flow.

---

# 1. High-Level System Overview

This project appears to be a **Toolbox / MCP Vehicle Management platform** used to manage:

- Employee data
- Vehicle ordering
- Spare vehicle ordering
- Vehicle allocation
- SAP handover/return/status reporting
- Vista confirmation/error processing
- Finance reports
- Loan agreements and annual statements
- Pricing matrix data
- Role/user/authorization management
- Authentication through ADFS/OIDC
- A Next.js/React frontend UI

The overall system is split into multiple backend Spring Boot services plus a frontend and gateway/auth services.

At a high level:

```text
User
 |
 | Browser
 v
toolboxui - Next.js / React / MUI frontend
 |
 | API calls
 v
Gateway / reverse proxy
 |
 +--> toolboxadfsbridge        - SSO / OIDC integration
 +--> toolboxadmin             - users, roles, authorizations
 +--> employee-data-service    - employee and vehicle scheme management
 +--> mcp-vehicle-order        - core MCP vehicle order lifecycle
 +--> mcp-vehicle-data-service - vehicle master data, pricing matrix, spare vehicle data
 +--> mcp-finance-service      - loan, finance, CVMS, reports
```

The backend services are secured using **OAuth2 Resource Server + JWT**, with local and non-local security configurations.

---

# 2. Main Services and Responsibilities

## A. `toolboxui` — Frontend Application

This is a **Next.js / React / TypeScript** frontend.

It contains pages/components for:

- Dashboard
- MCP order creation
- MCP order list
- Awaiting allocation
- Spare order
- Spare list
- User orders
- Employee management
- Scheme management
- Pricing matrix
- Vista bulk order
- Vista confirmation
- SAP status report
- SAP handover report
- SAP return report
- Symmetry report
- Loan agreement
- Annual statement
- Role management
- User management

Important frontend features:

- Uses **Redux** for form state:
  - `formSlices.ts` for MCP order form
  - `spareformSlice.ts` for spare order form
  - `uiSlice.ts`
- Uses **Material UI** and DataGrid.
- Has reusable components:
  - `CustomDatagrid`
  - `Filter`
  - `SortComponent`
  - `Pagination`
  - `FileUpload`
  - `Breadcrumb`
  - `Sidebar`
  - `Authorizations`
  - `InPageAuthorization`
- Has authorization utilities:
  - `getUserAuthorizationsDetails.ts`
  - `checkIfAcessPresent.ts`
  - `useButtons.tsx`
- Has protected and auth routes using Next.js app router:
  - `src/app/(protected)/...`
  - `src/app/(auth)/login`
  - `src/app/(unauth)/unauthorized`

### Interview explanation

You can say:

> The UI was built using Next.js with TypeScript and Material UI. It provided role-aware navigation and reusable table/filter/sort/pagination components. Most business screens were implemented as route-level pages under protected routes, and API calls were delegated to backend microservices through gateway endpoints. Authorization was enforced both on the backend and partially in the UI by filtering menu items and action buttons based on the user’s authorization codes.

---

## B. `toolboxadfsbridge` — ADFS / SSO Bridge

This service handles authentication with ADFS / OIDC.

Key files:

- `ToolBoxSsoController`
- `UserAuthenticationController`
- `UserService`
- `OAuth2SecurityConfig`

Main responsibilities:

- Authenticates user through OAuth2/OIDC.
- Extracts `OidcUser` claims.
- Creates/returns a `JlrUser`.
- Handles SSO callback.
- Likely stores token in cookie or redirects back to UI.

Important methods:

```java
toolBoxSSOHandler(...)
getOidcUserPrinciple(...)
getClaimsFromBean()
getJlrUserDetails()
getUserClaims()
```

### Flow

```text
User opens UI
 |
 v
Redirect to ADFS login
 |
 v
ADFS authenticates user
 |
 v
toolboxadfsbridge receives OIDC principal
 |
 v
Extracts claims and user details
 |
 v
Sets cookie/token or redirects to UI
```

### Interview explanation

> The ADFS bridge acts as the authentication integration layer. It integrates with enterprise SSO using OIDC, extracts claims from the authenticated principal, and provides user information to the rest of the Toolbox ecosystem.

---

## C. `toolboxadmin` — Admin, Roles, Users, Authorizations

This service manages:

- Users
- Roles
- Authorizations
- Role-authorizations mapping
- User-role mapping
- Authorization lists from JSON

Main packages:

- `controller`
- `service`
- `repository`
- `model`
- `dto`
- `config`
- `Exception`

Important controllers:

- `AuthorizationController`
- `RoleController`
- `UserController`
- `LogoutController`

Important services:

- `AuthorizationService`
- `RoleService`
- `UserService`

Important repositories:

- `AuthorizationRepository`
- `RoleRepository`
- `UserRepository`

Notable query:

```java
SELECT DISTINCT RA.AUTHORIZATION_CODE
FROM ROLE_AUTHORIZATIONS RA
JOIN USER_ROLES UR ON RA.ROLE_NAME=UR.ROLE_NAME
WHERE UR.USER_ID = :userId
```

This means:

```text
User ID
 -> user_roles
 -> role_name
 -> role_authorizations
 -> authorization_code
```

This gives all authorization codes available to a user.

### Authorization Flow

```text
User logs in
 |
 v
Frontend gets user identity
 |
 v
Frontend/backend calls toolboxadmin
 |
 v
toolboxadmin resolves user's roles
 |
 v
toolboxadmin resolves authorization codes for roles
 |
 v
UI uses these codes to show/hide routes/buttons
 |
 v
Backend APIs can also enforce access using Spring Security/JWT
```

### Interview explanation

> The admin service was responsible for RBAC. Users could be assigned roles, roles were mapped to authorization codes, and the UI consumed authorization lists to drive menu visibility and button-level access. The service also loaded baseline authorization definitions from JSON.

---

## D. `employee-data-service` — Employee and Scheme Management

This appears to be the root-level Spring Boot service under:

```text
src/main/java/com/jlr/toolbox/employeeservice
```

The user called it `employee-data-service`.

Main responsibilities:

- Employee CRUD
- Employee search
- HR file uploads:
  - daily
  - monthly
  - leavers
- MCP vehicle scheme management
- Employee filtering, pagination, sorting

Important controllers:

- `EmployeeController`
- `HrFileController`
- `McpVehicleSchemeController`

Important services:

- `EmployeeService`
- `HrFileService`
- `McpVehicleSchemeService`

Important repository:

- `EmployeeRepository`
- `McpVehicleSchemeRepository`

### Employee APIs

The employee service supports:

- Create employee
- Update employee
- Delete employee
- Get employee by CDSID
- Get employee by first name and last name
- Search employees
- Paginated/filterable employee list

Important repository methods:

```java
findByCdsId(String cdsId)
findByFirstNameAndLastName(String firstName, String lastName)
searchEmployees(String searchText)
findAllByPayrollNoIn(...)
findAllByCdsIdIn(...)
```

### HR File Upload Flow

```text
HR Excel file upload
 |
 v
HrFileController
 |
 v
HrFileService
 |
 v
EmployeeDataMapper
 |
 v
Map rows to Employee entities
 |
 v
Validate / collect errors
 |
 v
Save employees
 |
 v
Return FileResponse with success/error details
```

### Vehicle Scheme Management

Vehicle schemes likely define rules such as:

- Scheme
- Tag number
- Whether CDS ID is required
- Priority
- Template
- Monthly override

This is used by vehicle ordering/spare ordering.

### Interview explanation

> The employee service maintained employee master data and supported batch ingestion from HR files. It also managed MCP vehicle schemes, which were referenced during vehicle ordering to determine scheme-specific attributes such as tag numbers, templates, priority, and monthly override values.

---

## E. `mcp-vehicle-order` — Core Vehicle Order Service

This is the heart of the platform.

It manages the **vehicle order lifecycle**.

Main responsibilities:

- Create/update/cancel vehicle orders
- Return vehicle
- Spare order creation and assignment
- Order allocation
- Bulk upload orders from Excel
- Vista processing
- SAP handover/status/return reporting
- Sales report processing
- Symmetry report
- Vehicle data report
- User order search/update
- Order status calculation
- Order validations

Important controllers:

- `VehicleOrderController`
- `McpOrderController`
- `SpareOrderController`
- `OrderForVistaController`
- `SAPStatusController`
- `SAPHandoverController`
- `SapReturnController`
- `SpareToFCLGController`
- `SymmetryReportController`
- `McpSalesReportController`
- `ManheimDataProcessingController`
- `UserOrdersController`
- `VehicleOrderDetailController`

Important services:

- `VehicleOrderService`
- `McpOrderService`
- `SpareOrderService`
- `CalculateOrderStatusService`
- `McpBulkUploadService`
- `OrderForVistaService`
- `SAPHandoverService`
- `SAPStatusService`
- `SapReturnService`
- `SymmetryReportService`
- `VehicleDataReportService`
- `UserOredersService`

Important entities/models:

- `VehicleOrderDetail`
- `VehicleOrderUserDetail`
- `OrderFinanceDetail`
- `VehicleOrderOption`
- `DealerFittedOrderAccessory`
- `VehicleOrderMustFitOption`
- `OrderWorkflowActivity`
- `VehicleOrderHistory`
- `VehicleReorderHistory`
- `McpOrder`
- `McpLateOrder`
- `SapReturn`
- `SapHandover`

### Core Order Creation Flow

A likely create order flow:

```text
Frontend MCP order form
 |
 v
VehicleOrderController.saveOrder()
 |
 v
VehicleOrderService.createOrder()
 |
 +--> Run save validations
 |      - CDSID/payroll validation
 |      - existing order validation
 |      - vehicle availability validation
 |
 +--> Calculate order status
 |      CalculateOrderStatusService.calculateSaveOrderStatus()
 |
 +--> Save:
 |      - VehicleOrderDetail
 |      - VehicleOrderUserDetail
 |      - OrderFinanceDetail
 |      - VehicleOrderOption
 |      - DealerFittedOrderAccessory
 |      - MustFit options
 |      - Workflow activity
 |
 +--> Create order history
 |
 v
Return OrderRequestResponseDTO
```

### Order Update Flow

```text
Frontend edits order
 |
 v
VehicleOrderController.modifyOrder()
 |
 v
VehicleOrderService.updateOrder()
 |
 +--> Fetch current order
 +--> Run update validations
 |      - current status validation
 |      - changed employee existing order validation
 |
 +--> Calculate updated status
 |      CalculateOrderStatusService.calculateUpdateOrderStatus()
 |
 +--> Save updated details
 +--> Create history
 |
 v
Return updated DTO
```

### Cancel Order Flow

```text
Frontend cancel action
 |
 v
VehicleOrderController.cancelVehicleOrder()
 |
 v
VehicleOrderService.cancelVehicleOrder()
 |
 v
Update status to CANCELLED
 |
 v
Create history event
```

### Return Vehicle Flow

```text
Frontend return dialog
 |
 v
VehicleOrderController.returnVehicle()
 |
 v
VehicleOrderService.returnVehicle()
 |
 +--> Update return mileage/reg/date
 +--> Possibly call vehicle data service to create mileage/registration history
 +--> Update order status
 +--> Create order history
```

### Order Status Calculation

The `CalculateOrderStatusService` centralizes state transition logic.

Important methods:

```java
calculateSaveOrderStatus(...)
calculateUpdateOrderStatus(...)
calculateUploadOrderStatus(...)
forSaveSpareOrder(...)
forSalesReportStatus(...)
```

This service decides statuses like:

- Awaiting allocation
- Submit to build
- Ready to handover
- Cancelled
- Returned
- On lease
- Off lease
- etc.

Based on fields like:

- VIN
- Common order number
- Build period
- CI code
- Actual handover date
- Return date
- Order origin
- Spare indicator

### Validation Layer

There is a clean strategy-like validation structure:

Save validations:

- `CDSIDAndPayrollNumberRule`
- `IsOrderExistRule`
- `VehicleAvailabilityRule`

Update validations:

- `ChangedEmployeeOrderExistUpdateRule`
- `CurrentOrderStatusUpdateRule`

Upload validations:

- `EccoOrderNumberRule`
- `OrderExistForPayrollExcellRule`
- `ReturnRegNoRule`
- `TagNumberRule`

Interfaces:

```java
SaveOrderValidation
UpdateOrderValidation
StdOrderUploadValidation
RetOrderUploadValidation
```

### Interview explanation

> The core order service encapsulated the complete vehicle order lifecycle. It separated responsibilities into controller, service, repository, validation, mapping, and reporting layers. Order status transitions were centralized in `CalculateOrderStatusService`, while validations were implemented as pluggable rule classes. This made order creation, update, upload, cancellation, allocation, and return flows easier to maintain and test.

---

## F. `mcp-vehicle-data-service` — Vehicle Master Data, Pricing Matrix, Spare Vehicle Data

This service manages vehicle reference/master data.

Main responsibilities:

- Brand/model/model year/derivative data
- Exterior/interior colour options
- Chargeable options
- Must-fit options
- Dealer-fit options
- Derivative finance
- Pay grade derivative mappings
- Pricing matrix upload/report generation
- Vehicle data
- Vehicle mileage and registration history
- Spare vehicle reserve/unreserve
- Spare/reserved/status history
- SAP status report upload

Important controllers:

- `BrandController`
- `ModelController`
- `ModelYearController`
- `VDerivativeController`
- `ChargeableOptionsController`
- `MustFitOptionsController`
- `ExteriorColourOptionsController`
- `InteriorColourOptionController`
- `DealerFitOptionsController`
- `PayGradeController`
- `PricingMatrixController`
- `VehicleController`
- `SpareController`
- `VehicleMileageHistoryController`
- `VehicleRegistrationHistoryController`
- `VehicleStatusHistoryController`
- `ReservedHistoryController`

Important services:

- `PricingMatrixService`
- `VDerivativeService`
- `VehicleService`
- `SpareService`
- `PayGradeService`
- `ChargeableOptionsService`
- `MustFitOptionsService`
- `ExteriorColourOptionsService`
- `InteriorColourOptionService`
- `DealerFitOptionsService`
- `SapStatusReportService`

### Pricing Matrix Flow

```text
Pricing matrix Excel uploaded
 |
 v
PricingMatrixController.savePricingMatrixDetails()
 |
 v
PricingMatrixService.savePricingMatrixDetails()
 |
 +--> Parse workbook/sheets
 +--> Save brand details
 +--> Save model details
 +--> Save model year details
 +--> Save derivative details
 +--> Save finance details
 +--> Save exterior colours
 +--> Save interior colours
 +--> Save chargeable options
 +--> Save must-fit options
 |
 v
Return FileResponse
```

This pricing matrix data is then consumed by the UI and order service to populate:

- Brand
- Model
- Model year
- Derivative
- Exterior colour
- Interior colour
- Options
- Loan value
- Monthly cost
- Pay grade eligibility

### Vehicle Master Data Flow

```text
Order service needs derivative/colour/options
 |
 v
Calls vehicle-data-service
 |
 +--> VDerivativeController
 +--> ExteriorColourOptionsController
 +--> InteriorColourOptionController
 +--> ChargeableOptionsController
 +--> MustFitOptionsController
 |
 v
Returns available vehicle configuration data
```

### Spare Vehicle Flow

The service manages spare inventory/reservation:

```text
Create spare vehicle
 |
 v
SpareController.createSpare()
 |
 v
SpareService.createSpare()
 |
 v
Save VehicleSpare
 |
 v
Create SpareHistory
```

Reserve spare:

```text
Reserve spare request
 |
 v
SpareController.reserveSpare()
 |
 v
SpareService.reserveSpare()
 |
 +--> Find spare
 +--> Mark reserved
 +--> Create ReservedHistory
```

Unreserve/remove spare:

```text
Unreserve or remove assigned spare
 |
 v
SpareService.unreservedSpare()
SpareService.removeAssignedSpare()
 |
 v
Update spare and create history
```

### Interview explanation

> The vehicle data service acted as the vehicle master-data service. It owned pricing matrix information, derivatives, options, colours, finance values, pay-grade mappings, and vehicle/spare master records. The order service depended on it to validate and enrich orders with vehicle configuration and history information.

---

## G. `mcp-finance-service` — Finance, Loan, CVMS and Reports

This service handles financial data and report generation.

Main responsibilities:

- Loan agreement report generation
- Annual statement report generation
- CVMS report upload
- JLR net payments report
- Vehicle order loan history
- Vehicle order loan value history

Important controllers:

- `LoanAgreementReportController`
- `AnnualStatementReportController`
- `CvmsReportController`
- `JLRNetPaymentsReportController`
- `VehicleLoanHistoryController`

Important services:

- `LoanAgreementReportService`
- `AnnualStatementReportService`
- `CvmsReportService`
- `JLRNetPaymentsReportsService`
- `VehicleLoanHistoryService`

Important report templates:

```text
reports/Annual_Statement_report.jrxml
reports/loan_agreement_report.jrxml
reports/loan_agreement_report_above_60k.jrxml
```

This suggests the service uses **JasperReports**.

### Loan Agreement Flow

```text
Frontend clicks download loan agreement
 |
 v
LoanAgreementReportController.downloadLoanPlanReport(orderId)
 |
 v
LoanAgreementReportService.generateLoanPlanReport(orderId)
 |
 +--> Fetch order details from mcp-vehicle-order service
 +--> Fetch registration number
 +--> LoanAgreementDataExtractor.extract()
 +--> LoanAgreementCalculationHelper.performCalculations()
 +--> LoanAgreementParameterMapper.map()
 +--> Fill Jasper report template
 |
 v
Return PDF/byte[]
```

### Annual Statement Flow

```text
Frontend downloads annual statement
 |
 v
AnnualStatementReportController
 |
 v
AnnualStatementReportService.generateAnnualStatementReport()
 |
 v
Return byte[] report
```

### CVMS Upload Flow

```text
CVMS file uploaded
 |
 v
CvmsReportController.uploadCvmsReport()
 |
 v
CvmsReportService.processAndSaveCvmsReport()
 |
 +--> Parse Excel
 +--> Map rows
 +--> Save report records
 |
 v
Return FileResponse
```

### JLR Net Payments Flow

```text
Upload net payments report
 |
 v
JLRNetPaymentsReportController.saveJlrNetPaymentDetails()
 |
 v
JLRNetPaymentsReportsService.saveJlrNetPaymentDetails()
 |
 v
Save records
```

View report:

```text
Frontend table request with filters
 |
 v
getNetPaymentsReport(filterPageSortCriteria)
 |
 v
FiltrationAndPaginationRepository
 |
 v
Return paginated data
```

### Interview explanation

> The finance service handled financial reporting and loan-related workflows. It generated Jasper-based loan agreements and annual statements, maintained vehicle loan history, and processed finance-related uploads like CVMS and net payments reports. It also called the vehicle order service to fetch order data needed for report generation.

---

## H. `toolboxlocalgw` — Local Gateway / Reverse Proxy

This is a Spring Cloud Gateway-style service.

Important class:

- `MyRewriteLocationResponseHeaderGatewayFilterFactory`

Responsibilities:

- Reverse proxy routing
- Rewrite `Location` headers
- Support local gateway behavior
- Map internal service hostnames to external hostnames

Important methods:

```java
fixedLocation(...)
rewriteLocation(...)
apply(...)
filter(...)
```

### Interview explanation

> The local gateway acted as a reverse proxy, primarily useful for local or routed environments. It included a custom gateway filter to rewrite `Location` response headers so redirects from internal service URLs could be translated to externally accessible URLs.

---

# 3. Cross-Cutting Architecture Patterns

## A. Spring Boot Microservices

Each backend service follows a familiar layered structure:

```text
Controller
 |
 v
Service
 |
 v
Repository
 |
 v
Database
```

With additional layers:

```text
DTOs
Models/Entities
Exceptions
Config
Utils
Validation
```

---

## B. Security

Most backend services include:

- `JwtConverter`
- `OAuth2SecurityConfig`
- `OAuth2SecurityConfigLocal`

This implies:

- Non-local profiles use OAuth2 Resource Server with JWT validation.
- Local profile may permit CORS and easier local testing.
- JWT authorities are probably extracted from a claim using `JwtConverter`.

Security flow:

```text
ADFS/OIDC login
 |
 v
JWT/access token
 |
 v
Frontend includes token/cookie
 |
 v
Backend validates JWT
 |
 v
JwtConverter maps claims to Spring authorities
 |
 v
Controller/service executes if authorized
```

---

## C. Filtering, Pagination and Sorting

Several services share a custom generic filtering framework:

- `FilterCriteria`
- `SortCriteria`
- `FilterOperations`
- `SortOperations`
- `FiltrationAndPaginationRepository`
- `FiltrationAndPaginationRepositoryImpl`
- `FiltrationSpecificationUtil`
- `PaginationUtil`
- `SortingUtil`

This is used in:

- Employee service
- Vehicle order service
- Finance service
- Vehicle data service

### Flow

```text
Frontend sends filterPageSortCriteria
 |
 v
Controller receives query param
 |
 v
Service parses JSON criteria
 |
 v
Repository.findAllWithCriteria(...)
 |
 v
FiltrationSpecificationUtil creates JPA Specification
 |
 v
PaginationUtil creates Pageable
 |
 v
SortingUtil creates Sort
 |
 v
Database query executed
 |
 v
FiltrationAndPaginationResultDTO returned
```

### Interview explanation

> A reusable filtering and pagination abstraction was implemented across services using Spring Data JPA Specifications. The frontend sends a serialized filter/sort/page criteria object, and backend utilities convert that into predicates, pageable queries, and sort orders.

---

## D. Exception Handling

Each service has global exception handling.

Examples:

- `GlobalExceptionHandler`
- `ExceptionHandlerControllerAdvice`

They handle:

- Bad request
- Conflict
- Resource not found
- Validation errors
- JSON processing errors
- IO errors
- Runtime/generic exceptions
- Transaction exceptions

### Interview explanation

> Each microservice had centralized exception handling using `@ControllerAdvice`, allowing consistent error response formatting and HTTP status mapping across APIs.

---

## E. File Upload Processing

Several services process Excel/CSV uploads:

### Employee service

- HR Daily
- HR Monthly
- HR Leavers

### Vehicle order service

- MCP order upload
- Sales report upload
- Vista CON CSV
- Vista errors XLSX
- SAP validation
- Manheim data

### Vehicle data service

- Pricing matrix upload
- SAP status report upload

### Finance service

- CVMS report upload
- JLR net payments upload

Typical file flow:

```text
MultipartFile upload
 |
 v
Controller endpoint
 |
 v
Service validates file type
 |
 v
Apache POI parses Excel/CSV
 |
 v
Rows mapped to DTOs/entities
 |
 v
Validation errors collected
 |
 v
Valid records saved
 |
 v
FileResponse returned with status/errors
```

---

## F. Reporting

Reports are generated as `byte[]` responses.

Used in:

- Finance:
  - Loan agreement
  - Annual statement
- Vehicle order:
  - SAP handover
  - SAP status
  - SAP return
  - Symmetry
  - Vehicle data report
  - Spare to FCLG
  - Vista Excel report
- Vehicle data:
  - Pricing matrix report

The frontend likely downloads these blobs.

---

# 4. Major Business Domains

## 1. User and Authorization Domain

Owned by:

- `toolboxadmin`
- `toolboxadfsbridge`
- `toolboxui`

Key concepts:

- User
- Role
- Authorization code
- SSO claims
- UI route/action authorization

---

## 2. Employee Domain

Owned by:

- `employee-data-service`

Key concepts:

- Employee
- CDS ID
- Payroll number
- First name / last name
- HR files
- Vehicle scheme

---

## 3. Vehicle Ordering Domain

Owned by:

- `mcp-vehicle-order`

Key concepts:

- Vehicle order
- User details
- Finance details
- Options
- Dealer fitted accessories
- Must-fit options
- Workflow activity
- Order history
- Reorder history
- Allocation
- Cancellation
- Return
- Status lifecycle

---

## 4. Vehicle Master Data Domain

Owned by:

- `mcp-vehicle-data-service`

Key concepts:

- Brand
- Model
- Model year
- Derivative
- Exterior colour
- Interior colour
- Chargeable options
- Dealer-fit options
- Must-fit options
- Pay grade mapping
- Pricing matrix
- Vehicle spare
- Vehicle mileage history
- Vehicle registration history
- Vehicle status history

---

## 5. Finance Domain

Owned by:

- `mcp-finance-service`

Key concepts:

- Loan agreement
- Annual statement
- Loan value history
- CVMS report
- Net payments report
- Jasper report templates

---

# 5. End-to-End Example Flow: Creating an MCP Order

This is one of the most important interview flows.

```text
1. User logs into Toolbox UI using ADFS SSO.

2. toolboxadfsbridge authenticates the user and exposes claims/user info.

3. toolboxui retrieves user authorization codes from toolboxadmin.

4. User opens MCP Order form.

5. UI fetches dropdown/reference data:
   - brands
   - models
   - model years
   - derivatives
   - colours
   - chargeable options
   - must-fit options
   from mcp-vehicle-data-service.

6. UI fetches employee details from employee-data-service using CDSID/name.

7. User submits the vehicle order.

8. toolboxui calls mcp-vehicle-order VehicleOrderController.saveOrder.

9. VehicleOrderService:
   - validates employee/CDSID/payroll
   - validates existing orders
   - validates vehicle availability
   - calculates order status
   - maps request DTO to entities
   - saves vehicle order details
   - saves user details
   - saves finance details
   - saves selected options
   - saves dealer fitted accessories
   - saves workflow activity
   - creates order history

10. Response returned to UI.

11. Order becomes visible in order list / allocation list depending on status.
```

---

# 6. End-to-End Example Flow: Pricing Matrix Upload

```text
1. Admin opens Pricing Matrix screen in UI.

2. User uploads Excel file.

3. toolboxui calls mcp-vehicle-data-service PricingMatrixController.

4. PricingMatrixService parses the Excel file using Apache POI.

5. It extracts:
   - brands
   - models
   - model years
   - derivatives
   - derivative finance
   - exterior colours
   - interior colours
   - chargeable options
   - must-fit options

6. Data is saved into corresponding tables.

7. The UI can now use this master data when creating vehicle orders.
```

---

# 7. End-to-End Example Flow: Loan Agreement Download

```text
1. User opens Loan Agreement screen.

2. UI calls mcp-finance-service with order ID.

3. LoanAgreementReportService fetches order details from mcp-vehicle-order.

4. Service extracts loan agreement data.

5. Calculation helper computes loan/monthly values.

6. Parameter mapper maps DTO values to Jasper parameters.

7. Jasper template is filled.

8. PDF/byte[] returned to UI.

9. UI downloads the report.
```

---

# 8. End-to-End Example Flow: Spare Order Assignment

```text
1. User opens Spare List / Spare Order screen.

2. UI fetches spare vehicle/order data.

3. User assigns spare order to employee.

4. mcp-vehicle-order SpareOrderController receives assignment request.

5. SpareOrderService:
   - fetches employee details
   - saves personal details for order
   - updates spare order details
   - calculates relevant status

6. mcp-vehicle-data-service may manage actual spare vehicle reservation state.

7. Histories are created for spare/reserved events.
```

---

# 9. Important Technical Talking Points for Interview

You can confidently say:

## Microservices

> The application was split into domain-oriented Spring Boot microservices: admin/RBAC, employee management, vehicle ordering, vehicle data, finance, SSO bridge, and UI. Each service owned its own domain logic and data access layer.

## Security

> Authentication was handled through ADFS/OIDC and backend services were secured as OAuth2 resource servers using JWT validation. A custom JWT converter mapped token claims into Spring Security authorities.

## RBAC

> Role-based access was maintained in the admin service. Roles mapped to authorization codes, users mapped to roles, and the UI used these authorization codes for route and action-level access.

## Dynamic Search

> We implemented reusable filtering, sorting, and pagination utilities using JPA Specifications. This enabled the frontend to send dynamic filter criteria without needing separate APIs for each query variant.

## File Processing

> Several business workflows were file-driven. We used multipart upload endpoints and Apache POI to parse Excel files, collect row-level validation errors, and return structured `FileResponse` objects.

## Reporting

> Report generation was handled by backend services returning byte arrays. The finance service used JasperReports templates for loan agreements and annual statements.

## Validation Design

> Vehicle order validation used a rule-based approach with interfaces like `SaveOrderValidation` and `UpdateOrderValidation`, making individual validation rules modular and testable.

## State Management

> Order status calculation was centralized in `CalculateOrderStatusService`, avoiding duplicated status transition logic across create, update, upload, sales report, and spare order flows.

---

# 10. The Architecture in One Interview-Friendly Paragraph

You can use this:

> This project was a microservices-based internal vehicle management platform for MCP vehicle ordering. The frontend was built in Next.js/React with reusable grid, filter, pagination, and authorization components. Authentication was integrated with ADFS/OIDC through a dedicated bridge service, while authorization was managed by an admin service using users, roles, and authorization codes. The core vehicle-order service handled order creation, updates, cancellation, returns, allocation, Vista processing, SAP reporting, spare orders, validations, and status transitions. Vehicle master data such as brands, derivatives, colours, options, pay-grade mappings, pricing matrix, and spare inventory was owned by a separate vehicle-data service. Employee information and HR file ingestion were handled by the employee service, and loan agreements, annual statements, CVMS, and payment reports were handled by the finance service. Across services, we reused Spring Boot layered architecture, JPA repositories, dynamic filtering/pagination, centralized exception handling, JWT security, and file/report processing patterns.

---

# 11. Potential Interview Questions You Should Prepare For

We can practice these one by one.

## Architecture

1. Why was this split into microservices?
2. What service owns vehicle order data?
3. What service owns pricing matrix data?
4. How does the UI know which buttons/routes to show?
5. How does authentication work?
6. How do services communicate?

## Vehicle Order

7. Explain the MCP order creation flow.
8. How is order status calculated?
9. What validations happen before saving an order?
10. How do you handle order updates?
11. How are vehicle returns handled?
12. How do you track order history?

## Data and Reporting

13. How does pricing matrix upload work?
14. How do SAP/Vista workflows work?
15. How do you generate loan agreement reports?
16. How do file upload errors get returned?

## Technical

17. How does filtering and pagination work?
18. How is exception handling standardized?
19. How is JWT converted to Spring Security authorities?
20. How would you improve this architecture?
21. How would you debug a failed order creation?
22. How do you ensure data consistency across services?

---

# 12. Things I Noticed That Are Worth Knowing

A few observations that could come up:

1. There is duplicated/common code across services:
   - Filtering/pagination utilities
   - Security config
   - RestTemplate utility
   - Exception classes

   Interview improvement point:

   > This could be extracted into a shared internal library to reduce duplication.

2. Some naming inconsistencies exist:
   - `swaggerConfig` lowercase in some services
   - `UserOredersService` typo
   - `applyValidatiion` typo
   - packages sometimes use uppercase like `Controller`, `Service`, `Repository` in finance service

   Interview-safe phrasing:

   > One area for improvement would be standardizing naming conventions and extracting common libraries.

3. The frontend `.next` build artifacts appear committed.

   Improvement point:

   > Generated `.next` files should typically be excluded from source control using `.gitignore`.

4. There are many synchronous service calls.

   Improvement point:

   > For cross-service workflows that do not need immediate consistency, events/message queues could improve resilience.

---

We can now go back and forth.

The best next step is probably one of these:

1. **Deep dive into MCP order creation/update/cancel/return flow**
2. **Deep dive into authentication and authorization**
3. **Prepare interview answer: “Explain your project architecture”**
4. **Service-by-service responsibility cheat sheet**
5. **Sequence diagrams for major flows**
6. **Potential improvements and tradeoffs**
7. **Mock interview Q&A**
