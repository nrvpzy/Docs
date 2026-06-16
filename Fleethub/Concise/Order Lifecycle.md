## MCP Order Lifecycle — Easy Memory Version

Think of MCP order lifecycle as:

```text
REQUEST → VALIDATE → CREATE → ALLOCATE → PROCESS → HANDOVER → RETURN → HISTORY
```

That is the full story.

---

# 1. What MCP Order Lifecycle Does

The MCP order lifecycle manages an employee vehicle order from start to finish.

It tracks:

- who the order is for
- what vehicle was selected
- what finance/options/accessories apply
- what status the order is in
- whether it needs allocation
- whether Vista/SAP processing is needed
- whether the vehicle was handed over
- whether the vehicle was returned
- what happened historically

In simple words:

> It is the end-to-end workflow engine for vehicle orders.

---

# 2. Main Flow to Remember

```text
Employee selected
   ↓
Vehicle configured
   ↓
Order created
   ↓
Business validations applied
   ↓
Initial status calculated
   ↓
Order saved
   ↓
Allocation / Vista / SAP workflows
   ↓
Vehicle handover
   ↓
Vehicle return
   ↓
History recorded
```

---

# 3. Step-by-Step

## Step 1: Employee/User Selected

The order starts with an employee or user.

The system needs details like:

- CDS ID
- name
- payroll number
- email/phone
- pay grade
- address
- tag number

Purpose:

> Know who the vehicle order belongs to.

---

## Step 2: Vehicle Configured

The user selects vehicle information:

- brand
- model
- model year
- derivative
- exterior colour
- interior colour
- CI code
- options
- dealer-fitted accessories

Purpose:

> Know what vehicle is being ordered.

---

## Step 3: Order Created

The UI sends the full order request to the order service.

The order service receives:

- employee details
- vehicle details
- finance details
- options/accessories
- workflow fields

Purpose:

> Capture the order in the system.

---

## Step 4: Validations Applied

Before saving, the system validates:

### Employee validation

> Is employee/CDS/payroll data valid?

### Duplicate order validation

> Does this employee already have an active order?

### Vehicle availability validation

> Is this vehicle/common order number already used or reserved?

Purpose:

> Prevent bad orders before they enter the workflow.

---

## Step 5: Status Calculated

The system calculates initial order status.

Status depends on fields like:

- VIN present?
- common order number present?
- build period present?
- CI code changed?
- handover/return dates?

Examples:

- awaiting allocation
- submit to build
- ready to handover
- cancelled
- returned

Purpose:

> Tell operations what should happen next.

---

## Step 6: Order Saved

The system saves different parts of the order:

- main order details
- employee/user details
- finance details
- selected options
- dealer-fitted accessories
- must-fit options
- workflow activity
- history

Purpose:

> Store the complete operational order record.

---

## Step 7: Allocation

If the order needs a vehicle assignment, it goes to allocation.

Operations users can:

- view orders awaiting allocation
- filter by derivative/status
- save allocation decision

Purpose:

> Match the order with an available vehicle/order.

---

## Step 8: Vista / SAP Processing

The order may go through external operational processes.

### Vista

Used for things like:

- common order number updates
- Vista errors
- bulk export
- exclusion from Vista

### SAP

Used for:

- handover reports
- status/CON reports
- return reports
- SAP exclusions

Purpose:

> Keep order data aligned with external enterprise systems.

---

## Step 9: Handover

When vehicle is ready, handover-related dates/statuses are updated:

- dealer delivered date
- booked handover date
- actual handover date
- invoice date
- PDI status

Purpose:

> Track when the vehicle is delivered to the employee/customer.

---

## Step 10: Return

Eventually the vehicle is returned.

The return API captures:

- return mileage
- return registration number
- VIN/common order number
- return details

It also updates vehicle history.

Purpose:

> Close or progress the lifecycle after vehicle return.

---

## Step 11: History Recorded

Every important action creates history:

- order created
- order modified
- order allocated
- Vista update
- SAP exclusion
- order cancelled
- vehicle returned

Purpose:

> Make the lifecycle auditable.

---

# 4. One-Line Memory Hook

Remember this:

```text
MCP order lifecycle takes an employee vehicle request, validates it, saves it, moves it through allocation/Vista/SAP, tracks handover and return, and records history.
```

---

# 5. Interview Version

Say this:

> The MCP order lifecycle starts when an employee vehicle order is created. The service validates employee details, duplicate active orders, and vehicle availability. Then it calculates the initial order status and saves the order across order, user, finance, options, accessories, workflow, and history tables. After creation, the order moves through allocation, Vista and SAP processing, handover, and eventually vehicle return. At every major step, status is updated and order history is recorded so operations teams have a single source of truth.

---

# APIs

If asked **“What APIs were involved in the MCP order lifecycle?”**, answer like this:

---

## Concise Interview Answer

I worked on APIs around the complete vehicle order lifecycle — create, read, update, cancel, return, allocation, and history.

The main APIs were:

---

## 1. Create Order API

```text
POST /orders
VehicleOrderController.saveOrder()
```

Used to create a new MCP vehicle order.

It accepted employee details, vehicle details, finance details, options, and accessories.

Flow:

```text
validate employee/order/vehicle → calculate initial status → save order → create history
```

---

## 2. Get Order Details API

```text
GET /orders/{vehicle-order-id}
VehicleOrderController.getOrderById()
```

Used to fetch full order details for view/edit screens.

It returned combined order data:

```text
vehicle details + user details + finance + options + accessories + workflow
```

---

## 3. Update / Modify Order API

```text
PUT /orders/{vehicle-order-id}
VehicleOrderController.modifyOrder()
```

Used to modify an existing order.

Flow:

```text
fetch current order → validate status/update rules → compare changes → recalculate status → update order → create history
```

---

## 4. Cancel Order API

```text
PUT /orders/{vehicle-order-id}/cancel
VehicleOrderController.cancelVehicleOrder()
```

Used to cancel an order.

Flow:

```text
fetch order → set status CANCELLED → save → create history
```

---

## 5. Return Vehicle API

```text
PUT /orders/{vehicle-order-id}/return
VehicleOrderController.returnVehicle()
```

Used when a vehicle is returned.

It captured:

```text
return mileage + return registration + return details
```

Flow:

```text
update return fields → call vehicle history API → update status → create history
```

---

## 6. Order History API

```text
GET /orders/{vehicle_order_id}/history
VehicleOrderController.getOrderHistory()
```

Used to show audit trail.

It returned lifecycle events like:

```text
created, modified, cancelled, returned, allocation changes
```

---

## 7. Order List / Search API

```text
GET /orders?filterPageSortCriteria=...
McpOrderController.getAllOrders()
```

Used for order list screens.

Supported:

```text
dynamic filtering + sorting + pagination
```

---

## 8. Allocation APIs

```text
GET /orders/allocation
McpOrderController.getOrdersForAllocation()

GET /orders/allocation/{derivative}
McpOrderController.getOrdersAgainstDerivativesForAllocation()

POST /orders/allocation
McpOrderController.saveOrderAllocation()
```

Used by operations team to:

```text
view orders awaiting allocation → filter by derivative → save allocation decision
```

---

## 9. Bulk Upload API

```text
POST /orders/upload
McpOrderController.upload()
```

Used to upload order Excel files.

Flow:

```text
parse Excel → validate rows → save valid orders → return row-level errors
```

---

# Final 30-Second Version

Say this:

> The main lifecycle APIs were create order, get order, update order, cancel order, return vehicle, order history, order list/search, allocation, and bulk upload. Create validated employee/order/vehicle data, calculated status, saved order details, and created history. Update fetched current order, applied update validations, recalculated status, and saved changes. Cancel and return were lifecycle transition APIs. Allocation APIs helped operations view and assign orders. List APIs supported filtering and pagination, and history APIs made the lifecycle auditable.