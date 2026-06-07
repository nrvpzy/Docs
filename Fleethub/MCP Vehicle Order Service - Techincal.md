# MCP Vehicle Order Service — Technical Deep Dive

`mcp-vehicle-order` is the **core transactional and workflow service** for the vehicle order domain. Technically, it is a Spring Boot microservice exposing REST APIs for:

- vehicle order lifecycle
- order creation/update/cancel/return
- allocation
- bulk upload
- Vista integration workflows
- SAP handover/status/return workflows
- spare order management
- order/user search
- dashboard metrics
- reports/downloads
- file processing
- filtering, sorting, pagination

It is not just CRUD. It is a **workflow-heavy service** that receives user actions, file uploads, external process updates, and report requests, then updates order state, creates history, and serves operational views.

---

# 1. High-Level Technical Architecture

Inside `mcp-vehicle-order`, the architecture follows a layered Spring Boot structure:

```text
Controller Layer
   ↓
Service Layer
   ↓
Validation / Mapping / Utility Layer
   ↓
Repository Layer
   ↓
Database
```

Cross-cutting pieces:

```text
Security
Exception Handling
Filtering/Pagination Framework
File Upload Processing
Report Generation
RestTemplate-based service calls
Audit Handling
Status Calculation
```

---

# 2. Main Packages and Responsibilities

## 2.1 Controllers

Controllers expose REST endpoints.

Important controllers:

- `VehicleOrderController`
- `McpOrderController`
- `OrderForVistaController`
- `SAPHandoverController`
- `SAPStatusController`
- `SAPValidationController`
- `SapReturnController`
- `SpareOrderController`
- `SpareToFCLGController`
- `SymmetryReportController`
- `UserOrdersController`
- `VehicleOrderDetailController`
- `VehicleDataReportController`
- `McpSalesReportController`
- `ManheimDataProcessingController`
- `VehicleReorderHistoryController`

---

## 2.2 Services

Business logic lives in services.

Important services:

- `VehicleOrderService`
- `McpOrderService`
- `CalculateOrderStatusService`
- `McpBulkUploadService`
- `OrderForVistaService`
- `SAPHandoverService`
- `SAPStatusService`
- `SAPValidationService`
- `SapReturnService`
- `SpareOrderService`
- `SpareToFCLGService`
- `SymmetryReportService`
- `VehicleDataReportService`
- `McpSalesReportService`
- `UserOredersService`
- `ManheimDataProcessingService`
- `VehicleOrderDetailService`

---

## 2.3 Repositories

Repositories provide persistence access.

Important repositories:

- `VehicleOrderDetailRepository`
- `McpOrderRepository`
- `VehicleOrderUserDetailRepository`
- `OrderFinanceDetailRepository`
- `VehicleOrderOptionRepository`
- `DealerFittedOrderAccessoryRepository`
- `VehicleOrderMustFitOptionRepository`
- `OrderWorkflowActivityRepository`
- `VehicleOrderHistoryRepository`
- `VehicleReorderHistoryRepository`
- `McpLateOrderRepository`
- `OrderForVistaRepository`
- `SAPHandoverRepository`
- `SapReturnRepository`
- `SymmetryReportRepository`

---

## 2.4 Validation Package

The order service uses a rule-based validation design.

Interfaces:

- `SaveOrderValidation`
- `UpdateOrderValidation`
- `StdOrderUploadValidation`
- `RetOrderUploadValidation`

Rules:

- `CDSIDAndPayrollNumberRule`
- `IsOrderExistRule`
- `VehicleAvailabilityRule`
- `ChangedEmployeeOrderExistUpdateRule`
- `CurrentOrderStatusUpdateRule`
- `EccoOrderNumberRule`
- `OrderExistForPayrollExcellRule`
- `ReturnRegNoRule`
- `TagNumberRule`

This design keeps business rules modular and testable.

---

# 3. Core Domain Model

A vehicle order is split across multiple domain entities.

## 3.1 `VehicleOrderDetail`

Main transactional order record.

Represents:

- vehicle order ID
- common order number
- VIN
- ECCO order number
- brand/model/derivative
- colour selections
- CI code
- status
- dates
- handover/return details
- spare flags
- operational fields

This is the central order entity.

---

## 3.2 `VehicleOrderUserDetail`

Stores user/employee data linked to an order.

Includes things like:

- CDS ID
- first name
- last name
- phone
- email
- payroll number
- pay grade
- tag number
- address
- post code
- pickup/drop location

---

## 3.3 `OrderFinanceDetail`

Stores finance details:

- loan value
- monthly deduction
- duration
- exception template
- agreement/finance values

---

## 3.4 `VehicleOrderOption`

Stores selected chargeable options.

Composite ID:

- vehicle order ID
- option code

---

## 3.5 `DealerFittedOrderAccessory`

Stores dealer fitted accessories.

Composite ID:

- vehicle order ID
- accessory/code

---

## 3.6 `VehicleOrderMustFitOption`

Stores required must-fit options for a vehicle derivative.

---

## 3.7 `OrderWorkflowActivity`

Stores operational workflow milestones:

- accepted by sales
- call over
- dealer delivered
- PDI status
- handover dates
- invoice dates

---

## 3.8 `VehicleOrderHistory`

Stores lifecycle audit events.

Examples:

- order created
- order modified
- order cancelled
- vehicle returned
- Vista CON updated
- SAP excluded

---

## 3.9 `McpOrder`

This appears to be a read/query-oriented model used heavily for lists, reports, dashboard queries, and operational views.

`McpOrderRepository` contains many report/list queries.

---

# 4. Controller-by-Controller API Breakdown

Now let’s go through the APIs.

---

# 5. `VehicleOrderController` — Core Order Lifecycle APIs

This is the most important controller for vehicle order lifecycle.

## APIs

```text
saveOrder
modifyOrder
getOrderById
getOrderHistory
cancelVehicleOrder
returnVehicle
```

---

## 5.1 Create Order API

Controller method:

```text
saveOrder(@Valid @RequestBody OrderRequestResponseDTO orderRequestResponseDTO)
```

Service method:

```text
VehicleOrderService.createOrder(OrderRequestResponseDTO orderRequestResponseDTO)
```

### What it does

Creates a new MCP vehicle order.

### Request contains

The request DTO contains nested details for:

- vehicle order details
- user details
- finance details
- selected options
- dealer-fitted accessories
- scheme/pickup details
- order metadata

### Technical flow

```text
Request reaches controller
   ↓
Spring validates @Valid request body
   ↓
VehicleOrderService.createOrder()
   ↓
Save validation rules execute
   ↓
Initial order status calculated
   ↓
Entities persisted
   ↓
Order history created
   ↓
Response DTO returned
```

### Business validations

The create flow uses save validation rules:

#### `CDSIDAndPayrollNumberRule`

Validates employee identity data.

Checks likely include:

- CDS ID present/valid
- payroll number present/valid
- required user details not missing

#### `IsOrderExistRule`

Prevents duplicate active orders.

Uses repository methods like:

```text
existsByCdsIdAndStatusIn
existsByFirstNameAndLastNameAndStatusIn
```

Even though named `exists`, these return lists of active matching orders.

#### `VehicleAvailabilityRule`

Checks whether the selected vehicle/order/common order number is available.

Prevents allocating/reserving the same vehicle twice.

### Status calculation

Uses:

```text
CalculateOrderStatusService.calculateSaveOrderStatus()
```

The initial status may depend on:

- VIN present
- common order number present
- build period present
- CI code
- handover readiness
- whether spare order or normal order

### Persistence

The create API may persist:

- `VehicleOrderDetail`
- `VehicleOrderUserDetail`
- `OrderFinanceDetail`
- `VehicleOrderOption`
- `DealerFittedOrderAccessory`
- `VehicleOrderMustFitOption`
- `OrderWorkflowActivity`
- `VehicleOrderHistory`

### Interview answer

> The create order API accepted a complete order DTO, validated employee and vehicle eligibility, prevented duplicate active orders, calculated the initial order status, saved the order across multiple normalized entities, and created an order history event.

---

## 5.2 Modify Order API

Controller method:

```text
modifyOrder(Integer vehicleOrderId, OrderRequestResponseDTO orderRequestResponseDTO)
```

Service method:

```text
VehicleOrderService.updateOrder(Integer vehicleOrderId, OrderRequestResponseDTO orderRequestResponseDTO)
```

### What it does

Updates an existing vehicle order.

### Technical flow

```text
Controller receives vehicleOrderId + updated DTO
   ↓
Service fetches current order
   ↓
Update validations execute
   ↓
Compares current vs updated values
   ↓
Recalculates status
   ↓
Updates persisted entities
   ↓
Creates order history
   ↓
Returns updated DTO
```

### Update validations

#### `CurrentOrderStatusUpdateRule`

Checks if order can be modified in its current status.

For example, terminal states like:

- cancelled
- returned
- off lease

may restrict updates.

#### `ChangedEmployeeOrderExistUpdateRule`

Handles employee reassignment/change cases.

Checks:

- Did personal/user info change?
- If yes, does the new employee already have an active order?
- Avoids duplicate active orders through update flow.

### Status calculation

Uses:

```text
CalculateOrderStatusService.calculateUpdateOrderStatus(updatedDTO, currentOrder)
```

It considers:

- current status
- updated fields
- VIN
- common order number
- build period
- CI code changes
- order progression rules

### Interview answer

> The update API was more complex than create because it had to compare the incoming request with the current persisted order. It validated whether the current status allowed modification, checked employee-change duplicate scenarios, recalculated order status, updated related entities, and created history.

---

## 5.3 Get Order By ID API

Controller method:

```text
getOrderById(Integer vehicleOrderId)
```

Service method:

```text
VehicleOrderService.getOrderById(Integer vehicleOrderId)
```

### What it does

Returns full order details for a specific order.

### Likely response

```text
OrderDetailsResponseDto
```

### Aggregated data

The service likely combines:

- order detail
- user detail
- finance detail
- options
- dealer-fitted accessories
- must-fit options
- workflow activity
- possibly history or metadata

### Interview answer

> The get-order API returned a complete order view by aggregating data from multiple order-related entities instead of exposing only the core order table.

---

## 5.4 Get Order History API

Controller method:

```text
getOrderHistory(Integer vehicleOrderId)
```

Service method:

```text
VehicleOrderService.getOrderHistoryByVehicleOrderId(Integer vehicleOrderId)
```

Repository:

```text
VehicleOrderHistoryRepository.findByVehicleOrderId(Integer vehicleOrderId)
```

### What it does

Returns audit trail/history for an order.

### Interview answer

> This API gave operations users visibility into lifecycle events for an order, such as creation, modifications, cancellation, return, or file-based updates.

---

## 5.5 Cancel Vehicle Order API

Controller method:

```text
cancelVehicleOrder(Integer vehicleOrderId)
```

Service method:

```text
VehicleOrderService.cancelVehicleOrder(Integer vehicleOrderId)
```

### What it does

Cancels an active order.

### Technical flow

```text
Fetch order
   ↓
Validate exists
   ↓
Set status to CANCELLED
   ↓
Persist update
   ↓
Create history event
```

### Interview answer

> Cancellation was modeled as a lifecycle transition. It updated order status to cancelled and wrote an audit history event.

---

## 5.6 Return Vehicle API

Controller method:

```text
returnVehicle(Integer vehicleOrderId, ReturnVehicleRequestDto requestDto)
```

Service method:

```text
VehicleOrderService.returnVehicle(Integer vehicleOrderId, ReturnVehicleRequestDto requestDto)
```

### What it does

Handles vehicle return.

### Request contains

Likely:

- return mileage
- return registration number
- return date
- related comments/details

The service method:

```text
callVehicleHistoryEndpoint(vin, vehicleOrderId, returnMiles, returnRegNo, commonOrderNo)
```

shows that return flow creates vehicle history externally.

### Technical flow

```text
Fetch order
   ↓
Validate request
   ↓
Update return fields on order
   ↓
Call vehicle-data-service to create mileage/registration history
   ↓
Update status
   ↓
Create order history
```

### External dependency

Talks to:

```text
mcp-vehicle-data-service
```

to create vehicle mileage/registration history.

### Interview answer

> The return API updated return mileage and registration details, called the vehicle history endpoint in the vehicle-data service, updated lifecycle status, and created order history.

---

# 6. `McpOrderController` — Operational Order List, Allocation, Bulk Upload APIs

This controller handles broader order operations and list screens.

## APIs

```text
getAllOrders
getOrdersForAllocation
getOrdersAgainstDerivativesForAllocation
saveOrderAllocation
getAllAnnualOrderList
getAllLoanOrderList
getAllLateOrders
upload
```

---

## 6.1 Get All Orders API

Controller:

```text
getAllOrders(@RequestParam String filterPageSortCriteria)
```

Service:

```text
McpOrderService.getAllOrders(filterPageSortCriteria)
```

### What it does

Returns paginated/filterable list of all MCP orders.

### Technical feature

Uses dynamic filtering/pagination framework.

### Request

`filterPageSortCriteria` likely includes:

- filters
- sort fields
- page number
- page size
- AND/OR criteria

### Response

```text
FiltrationAndPaginationResultDTO<?>
```

### Interview answer

> This API powered the main order list screen. It supported dynamic filtering, sorting, and pagination so users could search large order datasets efficiently.

---

## 6.2 Get Orders for Allocation API

Controller:

```text
getOrdersForAllocation(filterPageSortCriteria)
```

Service:

```text
McpOrderService.getOrdersAllocation(filterPageSortCriteria)
```

### What it does

Returns orders ready/waiting for allocation.

### Business logic

Likely filters orders in allocation-relevant statuses such as:

- awaiting allocation
- submit to build
- missing VIN/CON
- derivative-based matching

### Interview answer

> This API powered the awaiting-allocation screen, allowing operations users to see orders requiring allocation decisions.

---

## 6.3 Get Orders Against Derivative for Allocation API

Controller:

```text
getOrdersAgainstDerivativesForAllocation(String derivative, String filterPageSortCriteria)
```

Service:

```text
McpOrderService.getOrdersAgainstDerivativesForAllocation(filterPageSortCriteria, derivative)
```

### What it does

Returns allocation orders filtered by derivative.

### Why derivative matters

Derivative is a specific vehicle configuration. Allocation often needs matching by derivative.

### Interview answer

> This API helped allocation teams find candidate orders for a specific vehicle derivative.

---

## 6.4 Save Allocation API

Controller:

```text
saveOrderAllocation(@Valid @RequestBody SaveAllocationOrderResponseDto dto)
```

Service:

```text
McpOrderService.saveAllocationOrder(dto)
```

### What it does

Saves allocation decision.

### Likely updates

- assigned vehicle/order
- common order number
- status
- allocation metadata
- workflow activity
- history

### Interview answer

> The allocation save API persisted the allocation decision and moved the order forward in the lifecycle.

---

## 6.5 Get Annual Order List API

Controller:

```text
getAllAnnualOrderList(filterPageSortCriteria)
```

Service:

```text
McpOrderService.getAllAnnualOrderList(filterPageSortCriteria)
```

### What it does

Returns annual statement/report-related order list.

This likely supports annual statement workflows in finance/UI.

---

## 6.6 Get Loan Order List API

Controller:

```text
getAllLoanOrderList(filterPageSortCriteria)
```

Service:

```text
McpOrderService.getAllLoanOrderList(filterPageSortCriteria)
```

### What it does

Returns orders relevant to loan agreement/report flow.

Finance service may use order details for loan reports, while this API powers UI list selection.

---

## 6.7 Get Late Orders API

Controller:

```text
getAllLateOrders(filterPageSortCriteria, groupAFilter, groupBFilter)
```

Service:

```text
McpOrderService.getAllLateOrders(groupAFilter, groupBFilter, filterPageSortCriteria)
```

Supporting model:

```text
Late_Order_Enum
```

### What it does

Returns orders considered late based on business filters.

### Business logic

`Late_Order_Enum` has:

- `Group_A_Filter`
- `Group_B_Filter`
- `getBFilterBasedOnAFilter`

This suggests late orders have grouped filter categories.

### Interview answer

> Late orders were grouped using business-defined filter categories, allowing operations teams to investigate delayed orders by reason or stage.

---

## 6.8 Bulk Upload API

Controller:

```text
upload(@RequestPart("file") MultipartFile file)
```

Service:

```text
McpBulkUploadService.saveExcelDetails(file)
```

### What it does

Processes bulk MCP order Excel upload.

### Technical flow

```text
Receive MultipartFile
   ↓
Validate Excel format
   ↓
Parse workbook/sheets
   ↓
Map rows to PayrollDeductionFormDetails
   ↓
Apply upload validations
   ↓
Map DTOs to order entities
   ↓
Save valid records
   ↓
Collect row-level errors
   ↓
Return FileResponse
```

### Upload validation rules

- `EccoOrderNumberRule`
- `OrderExistForPayrollExcellRule`
- `TagNumberRule`

For return uploads:

- `ReturnRegNoRule`

### Utility support

- `MapperUtil`
- `DateParser`
- `ExcelIUtil`
- `CSVtoExcelConverter`

### Interview answer

> The upload API allowed batch order creation/update from Excel. It parsed rows, validated each record, saved valid rows, and returned structured errors for invalid rows.

---

# 7. `OrderForVistaController` — Vista Workflow APIs

Vista-related APIs manage exports/imports for Vista downstream processing.

## APIs

```text
getBulkOrderForVista
getAllVistaOrderWithStatusInBuildOrAwaitingCON
updateVistaCon
updateVistaError
excelReport
excludeOrder
```

---

## 7.1 Get Bulk Order for Vista API

Controller:

```text
getBulkOrderForVista(filterPageSortCriteria)
```

Service:

```text
OrderForVistaService.getBulkOrderForVista(filterPageSortCriteria)
```

### What it does

Returns orders eligible for Vista bulk processing/export.

### Technical feature

Uses filtering/pagination.

---

## 7.2 Get Vista Orders With Status API

Controller:

```text
getAllVistaOrderWithStatusInBuildOrAwaitingCON(filterPageSortCriteria)
```

Service:

```text
OrderForVistaService.getAllVistaOrderWithStatusInBuildOrAwaitingCON(filterPageSortCriteria)
```

### What it does

Returns Vista-related orders in statuses like:

- in build
- awaiting CON

CON means common order number.

---

## 7.3 Update Vista CON API

Controller:

```text
updateVistaCon(@RequestPart("file") MultipartFile file)
```

Service:

```text
OrderForVistaService.updateVistaConFromCsv(file)
OrderForVistaService.updateVistaConForOrder(vistaConDTOList, errors)
```

### What it does

Processes a CSV file from Vista to update common order numbers and statuses against vehicle order IDs.

### Technical flow

```text
Upload CSV
   ↓
Convert/read CSV
   ↓
Map rows to VistaConDTO
   ↓
Find matching orders
   ↓
Update CON/common order number
   ↓
Update order status
   ↓
Save orders
   ↓
Create history
   ↓
Call vehicle post API if needed
   ↓
Return FileResponse
```

### External call

`callVehiclePostApi(...)` suggests after saving orders, the service may notify `mcp-vehicle-data-service` to create/update vehicle records.

---

## 7.4 Update Vista Error API

Controller:

```text
updateVistaError(@RequestPart("file") MultipartFile file)
```

Service:

```text
OrderForVistaService.updateVistaErrorsFromXlsx(file)
OrderForVistaService.updateVistaErrorForOrder(vistaErrorDTOList, errors)
```

### What it does

Processes Vista error Excel file and updates order error details.

### Business logic

If Vista rejects or flags order data, those errors need to be reflected in Toolbox.

---

## 7.5 Vista Excel Report API

Controller:

```text
excelReport(@RequestBody List<Integer> orderId)
```

Service:

```text
OrderForVistaService.generateExcelReport(orderIds)
```

### What it does

Generates an Excel export for selected orders.

### Supporting methods

- `getUserAndOption`
- `getDealerFit`
- `getMustFit`

This means report includes:

- user info
- options
- dealer-fit options
- must-fit options

---

## 7.6 Exclude Order API

Controller:

```text
excludeOrder(@RequestBody List<ExcludeResponseDto> orderId)
```

Service:

```text
OrderForVistaService.excludeOrder(requestDtos)
```

### What it does

Marks selected orders as excluded from Vista processing/export.

### Interview answer

> Vista APIs handled the operational exchange between Toolbox and Vista: selecting eligible orders, exporting them, processing CON updates, processing error files, excluding records, and creating history.

---

# 8. `SAPHandoverController` — SAP Handover APIs

## APIs

```text
getAllOrders
excelReport
excludeSapOrders
```

---

## 8.1 Get SAP Handover List API

Controller:

```text
getAllOrders(filterPageSortCriteria)
```

Service:

```text
SAPHandoverService.getSapHandoverList(filterPageSortCriteria)
```

### What it does

Returns orders eligible for SAP handover.

Uses dynamic filtering/pagination.

---

## 8.2 Generate Handover Report API

Controller:

```text
excelReport(@RequestBody List<Integer> orderId)
```

Service:

```text
SAPHandoverService.generateHandoverReport(orderId)
```

### What it does

Generates SAP handover Excel/report for selected orders.

Returns:

```text
byte[]
```

---

## 8.3 Exclude SAP Orders API

Controller:

```text
excludeSapOrders(@RequestBody List<ExcludeResponseDto> dtoList)
```

Service:

```text
SAPHandoverService.excludeSapOrder(dtoList)
```

### What it does

Excludes selected orders from SAP handover report/list.

---

# 9. `SAPStatusController` — SAP Status / CON Report APIs

## APIs

```text
getConOrderList
excelReport
```

---

## 9.1 Get CON Order List API

Controller:

```text
getConOrderList(filterPageSortCriteria)
```

Service:

```text
SAPStatusService.getConOrderList(filterPageSortCriteria)
```

### What it does

Returns orders for CON/SAP status reporting.

CON likely refers to common order number-related SAP processing.

---

## 9.2 Generate CON Report API

Controller:

```text
excelReport()
```

Service:

```text
SAPStatusService.generateConReport()
```

### What it does

Generates downloadable SAP CON report.

---

# 10. `SAPValidationController` — SAP Validation Upload API

API:

```text
updateVistaCon(@RequestPart("file") MultipartFile file)
```

Service:

```text
SAPValidationService.processSapValidationFile(file)
```

### What it does

Processes SAP validation file.

### Technical support

Methods:

- `detectHeaderPosition`
- `findHeaderColumn`
- `getCellValue`

This suggests SAP files may not have fixed header position, so service detects header dynamically.

### Interview answer

> SAP validation upload parsed Excel files with dynamic header detection, extracted relevant columns, validated content, and returned row-level processing results.

---

# 11. `SapReturnController` — SAP Return APIs

## APIs

```text
getEligibleSAPReturns
generateSapReturnReport
excludeSapReturn
```

---

## 11.1 Get Eligible SAP Returns API

Controller:

```text
getEligibleSAPReturns(filterPageSortCriteria)
```

Service:

```text
SapReturnService.getEligibleReturns(filterPageSortCriteria)
```

### What it does

Returns vehicles/orders eligible for SAP return processing.

---

## 11.2 Generate SAP Return Report API

Controller:

```text
generateSapReturnReport(@RequestBody List<Integer> orderIds)
```

Service:

```text
SapReturnService.generateEligibleSapReturnExcel(orderId)
```

### What it does

Generates Excel report for selected eligible SAP returns.

---

## 11.3 Exclude SAP Return API

Controller:

```text
excludeSapReturn(@RequestBody List<ExcludeResponseDto> dtoList)
```

Service:

```text
SapReturnService.excludeSapReturnOrder(dtoList)
```

### What it does

Excludes selected orders from SAP return process.

---

# 12. `SpareOrderController` — Spare Order APIs

## APIs

```text
getSpareOrderDetails
saveSpareAlongWithDetails
assignSpareOrderToUser
deleteUserDetails
```

---

## 12.1 Get Spare Orders API

Controller:

```text
getSpareOrderDetails(filterPageSortCriteria)
```

Service:

```text
SpareOrderService.getAllSpareOrders(filterPageSortCriteria)
```

### What it does

Returns filterable list of spare MCP orders.

---

## 12.2 Save Spare Along With Details API

Controller:

```text
saveSpareAlongWithDetails(@RequestBody SpareWithDetailsDTO spareWithDetailsDTO)
```

Service:

```text
SpareOrderService.saveSpareAlongWithDetails(spareWithDetailsDTO)
```

### What it does

Creates spare order and associated details.

### Status calculation

May use:

```text
CalculateOrderStatusService.forSaveSpareOrder(CON, buildPeriod)
```

---

## 12.3 Assign Spare Order to User API

Controller:

```text
assignSpareOrderToUser(@RequestBody AssignSpareVehicleOrderDetailsToUserDTO dto)
```

Service:

```text
SpareOrderService.assignSpareOrderToUser(dto)
```

### What it does

Assigns a spare vehicle order to a user/employee.

### Service support

```text
getEmployee(dto)
savePersonalDetailsForOrder(employeeDTO, vehicleOrderId)
```

This means it fetches employee data, then saves employee details against the spare order.

### External dependency

Likely talks to employee service to fetch employee details.

---

## 12.4 Delete User Details API

Controller:

```text
deleteUserDetails(@RequestParam Integer id)
```

Service:

```text
SpareOrderService.deleteUserDtails(vehicleOrderId)
```

### What it does

Removes assigned user details from spare order.

---

# 13. `SpareToFCLGController` — Spare to FCLG APIs

## APIs

```text
getAllOrders
excelReport
```

---

## 13.1 Get Spare to FCLG List API

Service:

```text
SpareToFCLGService.getSpareToFCLGList(filterPageSortCriteria)
```

### What it does

Returns list of spare orders relevant for FCLG process.

---

## 13.2 Generate Spare to FCLG Report API

Service:

```text
SpareToFCLGService.generateSpareToFCLGReport()
```

### What it does

Generates downloadable Excel/report.

---

# 14. `SymmetryReportController` — Symmetry Report APIs

## APIs

```text
getAllOrders
excelReport
```

---

## 14.1 Get Symmetry Report List API

Service:

```text
SymmetryReportService.getSymmetryReportList(filterPageSortCriteria)
```

Repository:

```text
McpOrderRepository.findValidVehicleSymmetryDetails(tenDaysAgo)
```

### What it does

Returns orders eligible for symmetry report.

### Business logic

Repository filters:

- status not in certain terminal states
- not spare
- payroll order origin
- handover/return date conditions around `tenDaysAgo`

---

## 14.2 Generate Symmetry Report API

Service:

```text
SymmetryReportService.generateSymmetryReport()
```

### What it does

Generates downloadable symmetry report.

---

# 15. `UserOrdersController` — User Order Search APIs

## APIs

```text
getUserOrders
updateVehicleOrderUserDetails
```

---

## 15.1 Get User Orders API

Controller:

```text
getUserOrders(cdsId, firstName, lastName)
```

Service:

```text
UserOredersService.getUserOrders(cdsId, firstName, lastName)
```

Repository:

```text
McpOrderRepository.findUserOrders(cdsId, firstName, lastName)
```

### What it does

Searches orders for a specific user by:

- CDS ID
- first name
- last name

### Business use

Used by support/operations to find employee’s orders.

---

## 15.2 Update Vehicle Order User Details API

Controller:

```text
updateVehicleOrderUserDetails(vehicleOrderIds, VehicleOrderUserDetailUpdateRequestDTO)
```

Service:

```text
UserOredersService.updateVehicleOrderUserDetails(vehicleOrderIds, updateRequestDTO)
```

### What it does

Bulk updates user details across selected vehicle orders.

### Interview answer

> This API supported operational corrections to employee/order user details, useful when employee data needed to be fixed across multiple orders.

---

# 16. `VehicleOrderDetailController` — Dashboard/Count APIs

## APIs

```text
getOrderStatusCount
getMonthlyOrderSummary
getSpareOrderCount
```

---

## 16.1 Order Status Count API

Controller:

```text
getOrderStatusCount(OrderStatus status, String condition)
```

Service:

```text
VehicleOrderDetailService.getOrderStatusCount(status, condition)
```

Repository:

```text
VehicleOrderDetailRepository.countByStatusAndCreatedDateBetween(...)
```

### What it does

Returns count of orders by status and time condition.

Condition examples:

- Today
- Week
- Month
- All

### Business use

Dashboard cards.

---

## 16.2 Monthly Order Summary API

Controller:

```text
getMonthlyOrderSummary(int year)
```

Service:

```text
VehicleOrderDetailService.getOrdersCountByMonth(year)
```

Repository:

```text
McpOrderRepository.getOrdersCountByMonth(year)
```

### What it does

Returns monthly order counts for a year.

Used for dashboard chart.

---

## 16.3 Spare Order Count API

Controller:

```text
getSpareOrderCount(String brand)
```

Service:

```text
VehicleOrderDetailService.getSpareOrderCount(brand)
```

Repository:

```text
findByIsSpareTrueAndCommonOrderNoIsNotNullAndBrandIgnoreCase
findByIsSpareTrueAndCommonOrderNoIsNotNull
```

### What it does

Returns spare order counts, optionally by brand.

---

# 17. `VehicleDataReportController`

API:

```text
generateJlrVehicleDataReport()
```

Service:

```text
VehicleDataReportService.generateJlrVehicleDataReport()
```

### What it does

Generates JLR vehicle data report.

### Supporting logic

```text
formatChargeableOptions
formatDealerFitOptions
```

### Repository logic

Uses `McpOrderRepository.findEligibleJlrVehicleDataReport(cutoffDateTime)`.

Filters orders that:

- have VIN
- not returned
- accepted by sales
- not certain statuses
- dealer delivered date criteria

---

# 18. `McpSalesReportController`

API:

```text
SalesOrderReport(@RequestPart("file") MultipartFile file)
```

Service:

```text
McpSalesReportService.McpSalesReportFromXlsx(file)
```

### What it does

Processes MCP sales report Excel.

### Technical flow

```text
Upload XLSX
   ↓
Parse cells into McpSalesReportDto
   ↓
Validate/collect errors
   ↓
Update vehicle/order/sales data
   ↓
Update derivative details if needed
   ↓
Calculate status using forSalesReportStatus
   ↓
Return FileResponse
```

### External dependency

`updateVehicleAndDerivativeDetails` likely calls vehicle-data-service to update vehicle/derivative records.

---

# 19. `ManheimDataProcessingController`

APIs:

```text
saveNewVehicleDataRecords(file)
saveReturnedVehicleDataRecords(file)
```

Service:

```text
ManheimDataProcessingService.saveNewVehicleDataRecords(file)
ManheimDataProcessingService.saveReturnedVehicleDataRecords(file)
```

### What it does

Processes Manheim data files for:

- new vehicle records
- returned vehicle records

### Business use

Ingest external vehicle data updates into the order lifecycle.

---

# 20. `VehicleReorderHistoryController`

API:

```text
getOrderHistory(vehicle_order_id)
```

Service likely:

```text
VehicleOrderService.getReOrderHistoryByVehicleOrderId(vehicleOrderId)
```

Repository:

```text
VehicleReorderHistoryRepository.findByVehicleOrderId(vehicleOrderId)
```

### What it does

Returns reorder history for an order.

---

# 21. Dynamic Filtering, Sorting, Pagination

This is a major technical feature.

Many APIs accept:

```text
filterPageSortCriteria
```

Used in:

- order list
- allocation list
- late orders
- Vista lists
- SAP handover list
- SAP return list
- spare orders
- symmetry report
- dashboard/report views

---

## 21.1 Important Classes

```text
FilterCriteria
SortCriteria
FiltrationAndPaginationDTO
FiltrationAndPaginationResultDTO
FilterOperations
SortOperations
FiltrationAndPaginationRepository
FiltrationAndPaginationRepositoryImpl
FiltrationSpecificationUtil
PaginationUtil
SortingUtil
```

---

## 21.2 Flow

```text
Frontend sends filterPageSortCriteria
   ↓
Service parses criteria
   ↓
FilterCriteria list created
   ↓
SortCriteria list created
   ↓
FiltrationSpecificationUtil builds JPA Specification
   ↓
PaginationUtil builds Pageable
   ↓
SortingUtil builds Sort
   ↓
Repository executes dynamic query
   ↓
Returns FiltrationAndPaginationResultDTO
```

---

## 21.3 `FiltrationSpecificationUtil`

Important methods:

```text
createSpecification
getPredicate
getValueAsPerDataType
```

### What it does

Converts generic filter conditions into JPA predicates.

Examples:

```text
status = AWAITING_ALLOCATION
brand LIKE JAGUAR
createdDate >= date
isSpare = true
```

It handles type conversion for:

- String
- Integer
- Boolean
- Enum
- Date/LocalDateTime
- possibly BigDecimal

---

## 21.4 Optimization Benefits

Instead of creating many APIs like:

```text
/orders/by-status
/orders/by-brand
/orders/by-status-and-date
/orders/by-vin
```

one API can handle multiple filters.

Benefits:

- fewer endpoints
- reusable backend framework
- consistent frontend filtering
- database-side pagination
- better admin search experience

---

# 22. File Upload Processing

The service has multiple file upload APIs.

## Upload types

- MCP bulk order upload
- MCP sales report upload
- Vista CON CSV
- Vista error XLSX
- SAP validation file
- Manheim new/returned vehicle data

---

## Common file processing pattern

```text
Receive MultipartFile
   ↓
Validate file type
   ↓
Parse using Apache POI / CSV reader
   ↓
Convert rows into DTOs
   ↓
Validate row-level data
   ↓
Collect errors
   ↓
Save valid updates
   ↓
Return FileResponse
```

---

## Utilities

### `ExcelIUtil`

- validate Excel format
- create Excel download headers
- sanitize data
- remove metadata

### `CSVtoExcelConverter`

- convert CSV to Excel workbook
- read CSV rows

### `DateParser`

- parse multiple date formats into `LocalDateTime`

### `MapperUtil`

Maps upload DTOs into entities:

- `VehicleOrderDetail`
- `VehicleOrderUserDetail`
- `OrderFinanceDetail`
- `OrderWorkflowActivity`
- `VehicleOrderOption`
- `DealerFittedOrderAccessory`

---

# 23. Status Calculation

Important service:

```text
CalculateOrderStatusService
```

Methods:

```text
calculateSaveOrderStatus
calculateUpdateOrderStatus
calculateUploadOrderStatus
forSaveSpareOrder
forSalesReportStatus
isCiCodeChanged
```

---

## Why centralized?

Order status can change through:

- create
- update
- upload
- allocation
- Vista updates
- sales report updates
- spare order creation
- return
- cancellation

Centralizing status avoids inconsistent rules.

---

## Status factors

Status can depend on:

- VIN
- common order number
- build period
- CI code changes
- confirmed dates
- handover dates
- return dates
- spare flag
- current status
- order origin

---

## Tests

There are tests for status behavior:

- save status awaiting allocation
- save status pre-holding pool
- save status ready to handover
- update with/without VIN/CON/build period
- CI code changed/same cases

This is a strong interview point.

---

# 24. Validation Architecture

Validation is modular.

## Save validations

```text
CDSIDAndPayrollNumberRule
IsOrderExistRule
VehicleAvailabilityRule
```

## Update validations

```text
CurrentOrderStatusUpdateRule
ChangedEmployeeOrderExistUpdateRule
```

## Upload validations

```text
EccoOrderNumberRule
OrderExistForPayrollExcellRule
ReturnRegNoRule
TagNumberRule
```

---

## Why this design is good

- avoids giant service methods
- separates rules by use case
- easy to unit test
- easy to add new rules
- improves readability

---

# 25. Exception Handling

Global exception handler:

```text
GlobalExceptionHandler
```

Handles:

- `BadRequestException`
- `ConflictException`
- `GoneException`
- `CustomIOException`
- `CustomJsonProcessingException`
- `ResourceNotFoundException`
- `NullPointerException`
- `MethodArgumentNotValidException`
- `TransactionSystemException`
- generic `Exception`

### Benefits

- consistent API error responses
- controllers stay clean
- maps business exceptions to HTTP statuses
- validation errors returned as maps/messages

---

# 26. Security

The service has:

```text
OAuth2SecurityConfig
OAuth2SecurityConfigLocal
JwtConverter
```

---

## Production-style security

`OAuth2SecurityConfig` configures Spring Security as an OAuth2 Resource Server.

Requests require a valid JWT.

---

## Token source

Method:

```text
cookieTokenResolver()
```

This means the service can extract JWT from cookie instead of only `Authorization` header.

---

## JWT conversion

`JwtConverter` converts JWT claims into Spring Security authentication.

---

## Local config

`OAuth2SecurityConfigLocal` likely allows local CORS/config differences for development.

---

# 27. Inter-Service Communication

The service talks to other microservices using:

```text
RestTemplateUtil
RestTemplate
HttpClient
```

Important methods:

```text
RestTemplateUtil.getJwt()
RestTemplateUtil.callMicroservice(url, method, responseType, requestBody)
```

### What it likely does

- gets current JWT from security context
- propagates token to downstream service
- calls microservice using RestTemplate

---

## Services it talks to

### `mcp-vehicle-data-service`

Used for:

- vehicle history creation during return
- exterior colour description
- interior colour description
- derivative/vehicle updates
- vehicle post API after Vista CON update
- mileage/registration history

### Employee service

Used for:

- assigning spare order to user
- employee lookup/details

### Finance service

Finance service likely calls this service, rather than this service calling finance, for loan reports.

---

# 28. API Optimization Considerations

## 28.1 Pagination everywhere

Large operational lists use:

```text
filterPageSortCriteria
```

This avoids returning huge datasets.

---

## 28.2 Database-side filtering

Filtering is pushed to database through JPA Specifications and queries.

Avoids fetching all orders and filtering in memory.

---

## 28.3 Indexing

Important indexes should exist on fields commonly filtered:

- vehicle_order_id
- status
- common_order_no
- vin
- cds_id
- first_name/last_name
- brand
- model
- derivative
- created_date
- order_confirmed_date
- actual_handover_date
- actual_return_date
- is_spare

Composite indexes useful for:

- `(status, created_date)`
- `(cds_id, status)`
- `(brand, model, derivative)`
- `(common_order_no)`
- `(vin)`

---

## 28.4 DTO projections

For list APIs, avoid returning full entities with all relationships.

Use DTOs/projections for:

- order list rows
- dashboard cards
- report lists

---

## 28.5 Avoid N+1 queries

For reports like Vista export, SAP handover, and JLR vehicle data report, related options/accessories are needed.

Optimization strategies:

- batch fetch by vehicle order IDs
- use repository methods like `findById_vehicleOrderIdIn`
- avoid fetching options one order at a time

The repository already has batch methods:

```text
findById_vehicleOrderIdIn
findById_VehicleOrderIdIn
findByVehicleOrderIdIn
```

---

## 28.6 File processing optimization

For large Excel files:

- stream rows where possible
- validate row by row
- collect errors efficiently
- avoid loading excessive data into memory
- batch database saves
- use transaction boundaries carefully

---

## 28.7 Report generation optimization

Reports returning `byte[]` can be expensive.

Strategies:

- generate only selected order IDs
- limit export size
- async generation for very large files
- stream response instead of holding full file in memory
- cache static lookup data

---

# 29. Spring Boot / Technology Stack

Based on the repository map, the service likely uses:

## Spring Boot Web

For REST controllers.

## Spring Data JPA

Repositories extend:

```text
JpaRepository
JpaSpecificationExecutor
JpaRepositoryImplementation
```

## Hibernate / JPA Criteria API

Used in dynamic filtering.

## Spring Security OAuth2 Resource Server

JWT validation.

## Spring Validation

`@Valid`, `MethodArgumentNotValidException`.

## Apache POI

Excel file parsing/generation.

Evidence:

- `Sheet`
- `Row`
- `Cell`
- `XSSFWorkbook`
- `HSSFWorkbook`

## OpenAPI / Swagger

`swaggerConfig.myCustomConfig()`

## ModelMapper

`ModelMapperConfig.modelMapper()`

## RestTemplate / Apache HttpClient

For inter-service calls.

## Kubernetes/Helm

`chart/` folder contains deployment templates.

## Docker

Dockerfile present.

---

# 30. Testing

Tests exist for:

## Status calculation

```text
CalculateOrderStatusServiceTest
```

Covers create/update status scenarios.

## Save validations

```text
SaveOrderValidationTests
```

Covers:

- invalid CDS ID
- invalid payroll number
- valid details
- duplicate order by CDS ID
- duplicate order by first/last name
- vehicle available/reserved

## Update validations

```text
UpdateOrderValidationTests
```

Covers:

- employee changed and order exists
- name changed and order exists
- no personal info change
- valid/invalid current status

### Interview point

> The service had unit tests around the most business-critical logic: status calculation and validation rules.

---

# 31. How to Explain the Technical Architecture in Interview

Use this:

> Technically, `mcp-vehicle-order` was a Spring Boot microservice exposing REST APIs for the vehicle order lifecycle and operational workflows. Controllers accepted requests for order creation, update, cancellation, return, allocation, file uploads, SAP/Vista workflows, dashboards, and reports. The service layer handled business logic, validation, status calculation, entity persistence, history creation, and downstream service calls. Repositories used Spring Data JPA, and list screens used a reusable filtering/sorting/pagination framework based on JPA Specifications. File-based workflows used Apache POI/CSV utilities, and reports returned byte arrays. The service was secured as an OAuth2 Resource Server using JWT validation and integrated with other services using RestTemplate.

---

# 32. Important Cross Questions

## Q1. What are the main API groups?

Answer:

- core order lifecycle APIs
- allocation APIs
- Vista APIs
- SAP APIs
- spare order APIs
- user order search APIs
- dashboard/count APIs
- report generation APIs
- file upload APIs

---

## Q2. How does create order work technically?

Answer:

> Controller receives `OrderRequestResponseDTO`, service runs save validations, calculates status, maps DTO to entities, saves order/user/finance/options/accessories/workflow data, creates history, and returns response.

---

## Q3. How does update differ from create?

Answer:

> Update fetches current order, checks whether current status allows modification, compares employee details, prevents duplicate active orders for changed user, recalculates status based on current and updated state, saves changes, and creates history.

---

## Q4. How is duplicate order prevented?

Answer:

> Through validation rules using repository queries by CDS ID and first/last name against active statuses.

---

## Q5. How does filtering work?

Answer:

> The UI sends `filterPageSortCriteria`; backend parses it into filter and sort criteria; `FiltrationSpecificationUtil` builds JPA predicates; pagination and sorting are applied; repository returns `FiltrationAndPaginationResultDTO`.

---

## Q6. How are file uploads handled?

Answer:

> Multipart files are parsed using Excel/CSV utilities, rows are mapped into DTOs, validations are applied row-wise, valid records are saved, errors are collected, and a `FileResponse` is returned.

---

## Q7. How does service communicate with other services?

Answer:

> Through RestTemplate utilities that propagate JWT and call downstream services such as vehicle-data-service and employee service.

---

## Q8. Why centralize status calculation?

Answer:

> Because order status changes from many flows: create, update, bulk upload, Vista, SAP, sales report, spare, cancellation, and return. Centralizing status avoids inconsistent lifecycle transitions.

---

## Q9. How would you optimize APIs?

Answer:

> Use pagination, DB-side filtering, indexes, DTO projections, batch fetch related data, avoid N+1 queries, use async processing for large reports/uploads, and cache reference data.

---

# 33. Final Technical Ownership Pitch

Use this as your polished answer:

> `mcp-vehicle-order` was the main workflow service for vehicle orders. Technically, it exposed REST APIs for order creation, modification, cancellation, return, allocation, Vista and SAP workflows, spare orders, dashboards, reports, and file uploads. The service used Spring Boot, Spring Data JPA, Spring Security OAuth2 Resource Server, validation, Apache POI for Excel processing, RestTemplate for service-to-service communication, and a custom reusable filtering/pagination framework based on JPA Specifications.
>
> The core design separated controllers, services, validation rules, repositories, utilities, and status calculation. Business-critical validations were implemented as rule classes, and order status transitions were centralized in `CalculateOrderStatusService`. For operational lists, APIs accepted dynamic filter/sort/page criteria and returned paginated results. For file workflows, the service parsed uploaded files, validated rows, updated orders, and returned structured errors. For downstream workflows, it generated Vista/SAP/Symmetry/JLR reports and created order history for auditability.