# Module Overview

**Module Name:** Customer Returns
**Internal Identifier:** customer-returns
**Purpose:** Manages return requests submitted by customers. A return record can contain multiple line items. Returns follow a defined status lifecycle. Approving a return triggers inventory restocking. Product and variant names are auto-populated from the database on creation when IDs are provided but names are omitted.

---

# Business Context

When a customer wishes to return one or more products, a return request is created and enters a `pending` state. Store staff review the request and either approve it (restocking inventory and recording a refund method) or reject it. Once fulfillment is resolved, the return can be marked `completed`. The module tracks who processed the return, when it was processed, and the total refund amount involved.

---

# Functional Scope

## Included

- Create, read, update, and soft-delete customer returns
- Multi-item returns (one return → many return items)
- Status lifecycle: `pending` → `approved` or `rejected` → `completed`
- Dedicated approve and reject action endpoints
- Inventory restock on approval (variant inventory and simple product stock)
- Auto-generation of `return_number` if not provided
- Auto-fill of product/variant names from the database when IDs are provided
- Filtering and pagination on the list endpoint
- Aggregate statistics endpoint
- Returns listing scoped to a specific customer
- All operations scoped to the authenticated company via JWT

## Excluded

- Refund payment processing (only refund method is tracked, not actual payment execution)
- Returning to `pending` from `approved` or `rejected`
- Bulk return creation or import

---

# Architecture Notes

- All endpoints require Bearer JWT. `company_id` is extracted from the token and applied as a query scope.
- `return_number` defaults to `RET-{unix_microseconds}` if not provided by the caller.
- `request_date` defaults to the current timestamp if not provided.
- Product and variant names are auto-filled from the database when an item has a `productId` or `variantId` but no corresponding name — preventing broken name snapshots.
- The approve action runs inventory restock inside a transaction. For variant items, it creates a `variant_inventory` row at `location_id=1` if one does not already exist. For simple products, it increments `products.stock` directly.
- Only `pending` returns can be approved or rejected. Attempting to act on a non-pending return returns HTTP 400.
- The list endpoint orders results by `request_date DESC`.

---

# Data Model

## Table: `customer_returns`

| Column           | Type         | Constraints / Notes                                                       |
|------------------|--------------|---------------------------------------------------------------------------|
| `id`             | uint         | Primary key, auto-increment                                               |
| `company_id`     | uint         | FK → `companies.id`; indexed; JSON key: `companyId`                       |
| `return_number`  | varchar(100) | Unique; not null; JSON key: `returnNumber`                                |
| `customer_id`    | *uint        | FK → `customers.id`; nullable; JSON key: `customerId`                     |
| `customer_name`  | varchar(255) | JSON key: `customerName`                                                  |
| `order_id`       | *uint        | FK → `sells.id`; nullable; JSON key: `orderId`                            |
| `order_number`   | varchar(100) | JSON key: `orderNumber`                                                   |
| `total_amount`   | float64      | Not null; default: `0`; JSON key: `totalAmount`                           |
| `status`         | varchar(50)  | One of: `pending`, `approved`, `rejected`, `completed`; default: `pending`|
| `request_date`   | timestamp    | Not null; ordered DESC; JSON key: `requestDate`                           |
| `processed_date` | *timestamp   | Nullable; set on approve or reject; JSON key: `processedDate`             |
| `refund_method`  | varchar(100) | Not null; one of: `cash`, `store_credit`, `original_payment`; JSON key: `refundMethod` |
| `notes`          | text         |                                                                           |
| `processed_by`   | varchar(255) | JSON key: `processedBy`                                                   |
| `created_at`     | timestamp    | JSON key: `createdAt`                                                     |
| `updated_at`     | timestamp    | JSON key: `updatedAt`                                                     |
| `deleted_at`     | timestamp    | Soft delete                                                               |

## Table: `customer_return_items`

| Column         | Type         | Constraints / Notes                              |
|----------------|--------------|--------------------------------------------------|
| `id`           | uint         | Primary key, auto-increment                      |
| `return_id`    | uint         | FK → `customer_returns.id`; not null             |
| `product_id`   | *uint        | FK → `products.id`; nullable                     |
| `product_name` | varchar(255) | Not null; JSON key: `productName`                |
| `variant_id`   | *uint        | FK → `product_variants.id`; nullable             |
| `variant_name` | varchar(255) | JSON key: `variantName`                          |
| `quantity`     | int          | Not null; default: `1`                           |
| `price`        | float64      | Not null; default: `0`                           |
| `reason`       | varchar(255) | Not null                                         |
| `created_at`   | timestamp    |                                                  |
| `updated_at`   | timestamp    |                                                  |

## Relationships

| Relationship                              | Type       | Notes                                        |
|-------------------------------------------|------------|----------------------------------------------|
| `CustomerReturn` → `Customer`             | Belongs To | Optional; nullable `customer_id`             |
| `CustomerReturn` → `Sell`                 | Belongs To | Optional; nullable `order_id`                |
| `CustomerReturn` → `CustomerReturnItems`  | Has Many   | Via `return_id`                              |
| `CustomerReturnItem` → `Product`          | Belongs To | Optional; nullable `product_id`              |
| `CustomerReturnItem` → `ProductVariant`   | Belongs To | Optional; nullable `variant_id`              |

---

# API Endpoints

| Method | Path                               | Auth       | Description                                              |
|--------|------------------------------------|------------|----------------------------------------------------------|
| GET    | `/customer-returns/`               | Bearer JWT | List returns with pagination, search, and filters        |
| GET    | `/customer-returns/:id`            | Bearer JWT | Get a single return with customer and items              |
| POST   | `/customer-returns/`               | Bearer JWT | Create a new return                                      |
| PUT    | `/customer-returns/:id`            | Bearer JWT | Update status, notes, and/or processedBy                 |
| POST   | `/customer-returns/:id/approve`    | Bearer JWT | Approve a pending return and restock inventory           |
| POST   | `/customer-returns/:id/reject`     | Bearer JWT | Reject a pending return                                  |
| DELETE | `/customer-returns/:id`            | Bearer JWT | Soft delete a return                                     |
| GET    | `/customer-returns/stats`          | Bearer JWT | Get aggregate counts and total refund amount             |
| GET    | `/customer-returns/customer/:customerId` | Bearer JWT | List all returns for a specific customer           |

---

# Request Payloads

## POST `/customer-returns/` — Create Return

**Content-Type:** `application/json`

```json
{
  "returnNumber": "",
  "customerId": 5,
  "customerName": "John Doe",
  "orderId": 3,
  "orderNumber": "INV-1700000000",
  "totalAmount": 59.98,
  "status": "pending",
  "requestDate": "2024-01-10T00:00:00Z",
  "refundMethod": "original_payment",
  "notes": "",
  "items": [
    {
      "productId": 2,
      "productName": "T-Shirt",
      "variantId": 5,
      "variantName": "Small / Red",
      "quantity": 2,
      "price": 29.99,
      "reason": "Wrong size"
    }
  ]
}
```

| Field          | Required | Notes                                                              |
|----------------|----------|--------------------------------------------------------------------|
| `returnNumber` | No       | Auto-generated as `RET-{unix_microseconds}` if empty              |
| `customerId`   | No       | Links to customer record                                           |
| `customerName` | No       |                                                                    |
| `orderId`      | No       | Links to original order                                            |
| `orderNumber`  | No       |                                                                    |
| `totalAmount`  | No       | Defaults to `0`                                                    |
| `status`       | No       | Default: `pending`; one of: `pending`, `approved`, `rejected`, `completed` |
| `requestDate`  | No       | Defaults to current timestamp                                      |
| `refundMethod` | Yes      | One of: `cash`, `store_credit`, `original_payment`                |
| `notes`        | No       |                                                                    |
| `items`        | No       | Array of return line items                                         |
| `items[].productId`   | No  | If provided without `productName`, name is auto-filled from DB |
| `items[].productName` | No  | Required on item if `productId` is not provided            |
| `items[].variantId`   | No  |                                                                |
| `items[].variantName` | No  | Auto-filled from DB if `variantId` is provided without name  |
| `items[].quantity`    | No  | Default: `1`                                                   |
| `items[].price`       | No  | Default: `0`                                                   |
| `items[].reason`      | Yes | Required per item                                              |

## PUT `/customer-returns/:id` — Update Return

**Content-Type:** `application/json`

```json
{
  "status": "completed",
  "notes": "Refund processed",
  "processedBy": "Admin"
}
```

## POST `/customer-returns/:id/reject` — Reject Return

**Content-Type:** `application/json`

```json
{
  "notes": "Return request rejected due to policy"
}
```

Notes field is optional. The approve endpoint takes no request body.

---

# Response Contracts

## GET `/customer-returns/` — 200 OK

```json
{
  "message": "Customer returns retrieved successfully",
  "data": [ /* array of return objects with nested customer and items */ ],
  "pagination": {
    "page": 1,
    "per_page": 10,
    "total": 15
  }
}
```

**Query parameters:**

| Param         | Type   | Default | Description                                                         |
|---------------|--------|---------|---------------------------------------------------------------------|
| `page`        | int    | 1       |                                                                     |
| `per_page`    | int    | 10      |                                                                     |
| `search`      | string |         | LIKE match on `return_number`, `customer_name`, `order_number`      |
| `status`      | string |         | One of: `pending`, `approved`, `rejected`, `completed`; `"all"` = no filter |
| `customer_id` | uint   |         | Filter by customer                                                  |

## GET `/customer-returns/:id` — 200 OK

```json
{
  "message": "Customer return fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "returnNumber": "RET-1700000000000",
    "customerId": 5,
    "customer": { },
    "customerName": "John Doe",
    "orderId": 3,
    "orderNumber": "INV-1700000000",
    "totalAmount": 59.98,
    "status": "pending",
    "requestDate": "2024-01-10T00:00:00Z",
    "processedDate": null,
    "refundMethod": "original_payment",
    "notes": "",
    "processedBy": "",
    "items": [
      {
        "id": 1,
        "returnId": 1,
        "productId": 2,
        "productName": "T-Shirt",
        "variantId": 5,
        "variantName": "Small / Red",
        "quantity": 2,
        "price": 29.99,
        "reason": "Wrong size",
        "createdAt": "...",
        "updatedAt": "..."
      }
    ],
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**404:** `{"error": "Customer return not found"}`

## POST `/customer-returns/` — 201 Created

```json
{
  "message": "Customer return created successfully",
  "data": { /* return object with customer and items */ }
}
```

## POST `/customer-returns/:id/approve` — 200 OK

```json
{
  "message": "Customer return approved and inventory restocked successfully",
  "data": { /* return object with status: "approved" */ }
}
```

## POST `/customer-returns/:id/reject` — 200 OK

```json
{
  "message": "Customer return rejected successfully",
  "data": { /* return object with status: "rejected" */ }
}
```

## DELETE `/customer-returns/:id` — 200 OK

```json
{
  "message": "Customer return deleted successfully"
}
```

## GET `/customer-returns/stats` — 200 OK

```json
{
  "message": "Statistics retrieved successfully",
  "data": {
    "total": 50,
    "pending": 10,
    "approved": 25,
    "rejected": 5,
    "completed": 10,
    "totalRefundAmount": 1249.95
  }
}
```

## GET `/customer-returns/customer/:customerId` — 200 OK

```json
{
  "message": "Customer returns retrieved successfully",
  "data": [ /* array of return objects ordered by requestDate DESC */ ]
}
```

---

# Validation Rules

| Field          | Rule                                                                    |
|----------------|-------------------------------------------------------------------------|
| `refundMethod` | Required; must be one of: `cash`, `store_credit`, `original_payment`   |
| `status`       | Must be one of: `pending`, `approved`, `rejected`, `completed`          |
| `returnNumber` | Must be unique; HTTP 409 returned if duplicate                          |
| `items[].reason` | Required per item                                                     |

---

# Business Logic Details

### Return Number Generation
If `returnNumber` is not provided (or is empty), the system auto-generates one as `RET-{unix_microseconds}`. If provided, it must be unique; a duplicate returns HTTP 409.

### Request Date Defaulting
If `requestDate` is not provided, it defaults to the current server timestamp at creation time.

### Product/Variant Name Auto-fill
On create, for each item: if `productId` is provided but `productName` is empty, the system fetches the product name from the database and fills it in. Same applies to `variantId` / `variantName`. This prevents broken name snapshots.

### Approve Action (`POST /customer-returns/:id/approve`)
- Only returns with `status = "pending"` can be approved; any other status returns HTTP 400.
- Inventory restock runs inside a database transaction.
- **Variant items:** Adds returned quantity to `variant_inventory`. If no `variant_inventory` row exists for the variant, a new row is created at `location_id = 1`.
- **Simple products:** Increments `products.stock` directly.
- Sets `processed_date` to the current timestamp and `status` to `"approved"`.

### Reject Action (`POST /customer-returns/:id/reject`)
- Only returns with `status = "pending"` can be rejected; any other status returns HTTP 400.
- No inventory changes occur on rejection.
- Sets `status` to `"rejected"` and `processed_date` to the current timestamp.
- Optional `notes` from the request body are saved.

### General Update (`PUT /customer-returns/:id`)
- Updates `status`, `notes`, and/or `processedBy`.
- Does not trigger inventory changes or status-transition guards (unlike the dedicated approve/reject endpoints).
- Invalid status value returns HTTP 400.

---

# Dependencies

| Module      | Reason                                                               |
|-------------|----------------------------------------------------------------------|
| `companies` | `company_id` scoping from JWT                                        |
| `customers` | `customer_id` FK; customer data preloaded on fetch                   |
| `orders`    | `order_id` FK references `sells.id` for the originating order        |
| `products`  | Product name lookup on creation; stock incremented on approval       |
| `product_variants` | Variant name lookup on creation; `variant_inventory` updated on approval |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` from the JWT is applied to all queries; users cannot access returns belonging to another company.
- No role-based access control is documented at the endpoint level beyond JWT authentication.

---

# Error Handling

| HTTP Status | Trigger                                                                                      |
|-------------|----------------------------------------------------------------------------------------------|
| 400         | Invalid request body; invalid status value; invalid refund method; attempt to approve/reject a non-pending return |
| 404         | Return not found by ID                                                                        |
| 409         | Duplicate `return_number`                                                                    |
| 500         | Database error; failure to approve return or restock inventory; failure to reject return      |

---

# Testing Notes

- Verify `return_number` auto-generation produces a unique `RET-{unix_microseconds}` string.
- Verify HTTP 409 on duplicate `return_number`.
- Verify product and variant names are auto-filled from DB when IDs are provided without names.
- Verify that only `pending` returns can be approved; test HTTP 400 for `approved`, `rejected`, and `completed` states.
- Verify that only `pending` returns can be rejected; test HTTP 400 for other states.
- Verify inventory restock after approve:
  - Variant item: `variant_inventory` quantity increases; new row created at `location_id=1` if missing.
  - Simple product: `products.stock` increments.
- Verify `processed_date` is set on approve and reject.
- Verify `status` filter on `GET /customer-returns/`: `"all"` returns all statuses, specific value filters correctly.
- Verify `search` LIKE matches on `return_number`, `customer_name`, and `order_number`.
- Verify `GET /customer-returns/stats` totals are consistent with actual return records.

---

# Open Questions / Missing Details

- The approve action creates a `variant_inventory` row at `location_id=1` if none exists. Whether `location_id=1` is a system default or a configurable value is not documented.
- Whether `GET /customer-returns/customer/:customerId` respects the company scope or returns returns across all companies for that customer is not explicitly stated (assumed to be scoped).
- The `PUT /customer-returns/:id` general update endpoint does not trigger status-transition guards. Whether status transitions via this endpoint are intended to be unrestricted is not clarified.
- No validation rules are documented for individual item fields beyond `reason` being required.
- Whether soft-deleted returns are excluded from stats is not specified.

---

# Change Log / Notes

- Source documentation does not include a version history.
- This file was standardized from the original module notes on 2026-03-31.
