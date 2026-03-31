# Module Overview

**Module Name:** Orders (Sells)
**Internal Identifier:** sells
**Purpose:** Manages sales orders. Internally, orders are modelled as "Sells". Captures customer, line items, shipping information, payment method, and financial totals. Stock is deducted transactionally on order creation and restored on deletion. Supports inline custom shipping addresses or saved customer addresses. All operations are scoped by `company_id`.

---

# Business Context

When a customer purchases products, an order (sell) is created capturing a full snapshot of the transaction. The system deducts stock for each line item as part of the creation transaction to prevent overselling. Financial fields (`total_cost`, `gross_profit`) are computed from cost prices snapshotted at sale time, enabling profitability tracking per order. Orders can be tracked through fulfillment and payment status independently of the top-level order status.

---

# Functional Scope

## Included

- Full CRUD for orders (with soft delete)
- Line items with cost-price snapshotting and profit calculation
- Transactional stock deduction on creation; stock restoration on deletion
- Three shipping address resolution strategies (inline, saved, customer default)
- Shipping snapshot fields stored directly on the order record
- Coupon and discount tracking
- Payment status and fulfillment status tracking
- Tracking number and carrier fields
- Invoice number auto-generation
- Order statistics endpoint
- Separate status update endpoint (`PATCH`)
- Date range and status/method filtering on the list endpoint
- All operations scoped to authenticated company via JWT

## Excluded

- Re-deducting or restoring stock on order update (PUT does not affect stock)
- Shipment record management (handled by the Shipping module, referenced via `OrderShipments`)
- Payment processing (payment status is tracked, not executed here)

---

# Architecture Notes

- The model is named `Sell` internally; the table is `sells` and the API path is `/sells/`.
- `company_id` is required on all queries and is extracted from the Bearer JWT.
- Invoice number format: `INV-{unix_timestamp}`. Auto-generated if not provided by the caller.
- `order_time` defaults to the current server timestamp if not provided.
- `customerName` is required for every order. `customerId` is optional.
- Shipping address has three resolution paths (in priority order):
  1. Inline custom address: both `shippingFullName` and `shippingAddressLine1` must be provided; `shippingCity` and `shippingCountry` are also required; type defaults to `"other"`.
  2. Saved address by `shippingAddressId`: the address must belong to the provided customer.
  3. Auto-selected customer default address: used when only `customerId` is provided.
- Stock deduction runs inside the creation DB transaction. `stock_deducted = true` is set on the record after successful deduction.
- PUT update does not re-deduct or restore stock, regardless of quantity changes.
- `unit_cost` on each order item is snapshotted from `product_variants.cost_price` (if variant) or `products.cost_price` (if simple product) at the time of order creation.
- `sells.total_cost` = SUM of `order_items.total_cost`; `sells.gross_profit` = `sells.amount` - `sells.total_cost`.
- Items accept both camelCase (`unitPrice`) and snake_case (`price`) field names as aliases.

---

# Data Model

## Table: `sells`

| Column                  | Type         | Constraints / Notes                                                           |
|-------------------------|--------------|-------------------------------------------------------------------------------|
| `id`                    | uint         | Primary key, auto-increment; JSON key: `id`                                   |
| `company_id`            | uint         | Not null; indexed; JSON key: `companyId`                                      |
| `invoice_no`            | varchar(100) | Unique; not null; JSON key: `invoiceNo`                                       |
| `order_time`            | timestamp    | Not null; JSON key: `orderTime`                                               |
| `customer_id`           | *uint        | FK → `customers.id`; nullable; JSON key: `customerId`                         |
| `customer_name`         | varchar(255) | Not null; JSON key: `customerName`                                            |
| `shipping_address_id`   | *uint        | FK → `shipping_addresses.id`; nullable; JSON key: `shippingAddressId`         |
| `shipping_full_name`    | varchar(255) | Snapshot; JSON key: `shippingFullName`                                        |
| `shipping_phone`        | varchar(50)  | Snapshot; JSON key: `shippingPhone`                                           |
| `shipping_email`        | varchar(255) | Snapshot; JSON key: `shippingEmail`                                           |
| `shipping_address_line1`| varchar(255) | Snapshot; JSON key: `shippingAddressLine1`                                    |
| `shipping_address_line2`| varchar(255) | Snapshot; JSON key: `shippingAddressLine2`                                    |
| `shipping_city`         | varchar(100) | Snapshot; JSON key: `shippingCity`                                            |
| `shipping_state`        | varchar(100) | Snapshot; JSON key: `shippingState`                                           |
| `shipping_postal_code`  | varchar(20)  | Snapshot; JSON key: `shippingPostalCode`                                      |
| `shipping_country`      | varchar(100) | Snapshot; JSON key: `shippingCountry`                                         |
| `shipping_address_type` | varchar(50)  | Snapshot; JSON key: `shippingAddressType`                                     |
| `method`                | varchar(100) | One of: `Cash`, `Card`, `Online`; default: `Cash`; JSON key: `method`        |
| `amount`                | float64      | Default: `0`; JSON key: `amount`                                              |
| `shipping_cost`         | float64      | Default: `0`; JSON key: `shippingCost`                                        |
| `shipping_method`       | varchar(100) | JSON key: `shippingMethod`                                                    |
| `coupon_id`             | *uint        | FK → `coupons.id`; nullable; indexed; JSON key: `couponId`                    |
| `coupon_code`           | string       | JSON key: `couponCode`                                                        |
| `discount`              | float64      | Default: `0`; JSON key: `discount`                                            |
| `status`                | varchar(50)  | One of: `Pending`, `Processing`, `Delivered`; default: `Pending`              |
| `stock_deducted`        | bool         | Not null; default: `false`; JSON key: `stockDeducted`                         |
| `payment_status`        | enum         | One of: `pending`, `paid`, `partially_paid`, `refunded`, `failed`; default: `pending`; JSON key: `paymentStatus` |
| `fulfillment_status`    | enum         | One of: `unfulfilled`, `processing`, `shipped`, `delivered`, `cancelled`; default: `unfulfilled`; JSON key: `fulfillmentStatus` |
| `tracking_number`       | varchar(100) | JSON key: `trackingNumber`                                                    |
| `carrier`               | varchar(100) | JSON key: `carrier`                                                           |
| `shipped_at`            | *timestamp   | Nullable; JSON key: `shippedAt`                                               |
| `delivered_at`          | *timestamp   | Nullable; JSON key: `deliveredAt`                                             |
| `total_cost`            | float64      | Default: `0`; SUM of `order_items.total_cost`; JSON key: `totalCost`          |
| `gross_profit`          | float64      | Default: `0`; `amount - total_cost`; JSON key: `grossProfit`                  |
| `notes`                 | text         | JSON key: `notes`                                                             |
| `created_at`            | timestamp    | JSON key: `createdAt`                                                         |
| `updated_at`            | timestamp    | JSON key: `updatedAt`                                                         |
| `deleted_at`            | timestamp    | Soft delete                                                                   |

## Table: `order_items`

| Column         | Type    | Constraints / Notes                                                      |
|----------------|---------|--------------------------------------------------------------------------|
| `id`           | uint    | Primary key, auto-increment                                              |
| `sell_id`      | uint    | FK → `sells.id`                                                          |
| `product_id`   | *uint   | FK → `products.id`; nullable                                             |
| `variant_id`   | *uint   | FK → `product_variants.id`; nullable                                     |
| `inventory_id` | *uint   | FK → `variant_inventory.id`; nullable; used to target a specific inventory row |
| `product_name` | string  | JSON key: `productName`                                                  |
| `variant_name` | string  | JSON key: `variantName`                                                  |
| `quantity`     | int     | JSON key: `quantity`                                                     |
| `unit_price`   | float64 | JSON key: `unitPrice`                                                    |
| `total_price`  | float64 | JSON key: `totalPrice`                                                   |
| `unit_cost`    | float64   | Default: `0`; cost_price snapshotted at sale time; JSON key: `unitCost`  |
| `total_cost`   | float64   | Default: `0`; `unit_cost × quantity`; JSON key: `totalCost`              |
| `deleted_at`   | timestamp | Soft delete                                                              |

## Relationships

| Relationship                  | Type       | Notes                                       |
|-------------------------------|------------|---------------------------------------------|
| `Sell` → `Customer`           | Belongs To | Optional; nullable `customer_id`            |
| `Sell` → `ShippingAddress`    | Belongs To | Optional; nullable `shipping_address_id`    |
| `Sell` → `Coupon`             | Belongs To | Optional; nullable `coupon_id`              |
| `Sell` → `OrderItems`         | Has Many   | Via `sell_id`                               |
| `Sell` → `OrderShipments`     | Has Many   | Via `sell_id`; managed by Shipping module   |

---

# API Endpoints

| Method | Path                      | Auth       | Description                                            |
|--------|---------------------------|------------|--------------------------------------------------------|
| GET    | `/sells/`                 | Bearer JWT | List orders with pagination, filters, and date range   |
| GET    | `/sells/:id`              | Bearer JWT | Get a single order with all relations                  |
| GET    | `/sells/invoice/:invoiceNo` | Bearer JWT | Get a single order by invoice number                 |
| POST   | `/sells/`                 | Bearer JWT | Create an order with transactional stock deduction     |
| PUT    | `/sells/:id`              | Bearer JWT | Partial update of order fields (no stock changes)      |
| PATCH  | `/sells/:id/status`       | Bearer JWT | Update order status only                               |
| DELETE | `/sells/:id`              | Bearer JWT | Soft delete order + restore stock if previously deducted |
| GET    | `/sells/stats`            | Bearer JWT | Get aggregate revenue and order count statistics       |

---

# Request Payloads

## POST `/sells/` — Create Order

**Content-Type:** `application/json`

```json
{
  "invoiceNo": "INV-2024-001",
  "orderTime": "2024-01-01T10:00:00Z",
  "customerId": 5,
  "customerName": "John Doe",
  "shippingAddressId": 3,
  "shippingFullName": "John Doe",
  "shippingPhone": "+1234567890",
  "shippingEmail": "john@example.com",
  "shippingAddressLine1": "123 Main St",
  "shippingAddressLine2": "",
  "shippingCity": "New York",
  "shippingState": "NY",
  "shippingPostalCode": "10001",
  "shippingCountry": "US",
  "shippingAddressType": "home",
  "method": "Card",
  "amount": 99.99,
  "shippingCost": 5.99,
  "shippingMethod": "Standard",
  "couponId": null,
  "couponCode": "",
  "discount": 0,
  "status": "Pending",
  "notes": "",
  "items": [
    {
      "productId": 2,
      "variantId": 5,
      "inventoryId": null,
      "productName": "T-Shirt",
      "variantName": "Small / Red",
      "quantity": 2,
      "unitPrice": 29.99
    }
  ]
}
```

> Items also accept snake_case aliases: `product_id`, `variant_id`, `inventory_id`, `price` (alias for `unitPrice`).

| Field          | Required | Notes                                                             |
|----------------|----------|-------------------------------------------------------------------|
| `customerName` | Yes      |                                                                   |
| `invoiceNo`    | No       | Auto-generated as `INV-{unix_timestamp}` if empty; must be unique |
| `orderTime`    | No       | Defaults to current timestamp                                     |
| `status`       | No       | One of: `Pending`, `Processing`, `Delivered`; default: `Pending` |
| `method`       | No       | One of: `Cash`, `Card`, `Online`; default: `Cash`                 |
| `items`        | No       | Array of order line items                                         |

## PUT `/sells/:id` — Partial Update

**Content-Type:** `application/json`

Any subset of the following fields: `invoiceNo`, `orderTime`, `customerId`, `customerName`, `method`, `amount`, `shippingCost`, `discount`, `status`, `notes`. Stock is not re-deducted or restored.

## PATCH `/sells/:id/status` — Update Status

**Content-Type:** `application/json`

```json
{
  "status": "Processing"
}
```

`status` is required; must be one of: `Pending`, `Processing`, `Delivered`.

---

# Response Contracts

## GET `/sells/` — 200 OK

**Query parameters:**

| Param         | Type   | Default | Description                                                             |
|---------------|--------|---------|-------------------------------------------------------------------------|
| `page`        | int    | 1       | Ignored if `limit` is set                                               |
| `per_page`    | int    | 10      |                                                                         |
| `limit`       | int    |         | If set, returns exactly N items with no pagination metadata             |
| `search`      | string |         | LIKE match on `customer_name`                                           |
| `status`      | string |         | One of: `Pending`, `Processing`, `Delivered`; `"all"` = no filter      |
| `method`      | string |         | One of: `Cash`, `Card`, `Online`; `"all"` = no filter                  |
| `customer_id` | uint   |         | Filter by customer                                                      |
| `start_date`  | string |         | Filter `order_time >=`; format: `2006-01-02`                            |
| `end_date`    | string |         | Filter `order_time <=` end of that day; format: `2006-01-02`           |

Paginated response:
```json
{
  "message": "Sells retrieved successfully",
  "data": [ /* array of sell objects */ ],
  "pagination": { "page": 1, "per_page": 10, "total": 100 }
}
```

With `limit` param (no pagination metadata):
```json
{
  "message": "Sells retrieved successfully",
  "data": [ /* array of sell objects */ ]
}
```

## GET `/sells/:id` — 200 OK

```json
{
  "message": "Sell fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "invoiceNo": "INV-1700000000",
    "orderTime": "2024-01-01T10:00:00Z",
    "customerId": 5,
    "customer": { },
    "customerName": "John Doe",
    "shippingAddressId": 3,
    "shippingAddress": { },
    "shippingFullName": "John Doe",
    "shippingPhone": "+1234567890",
    "shippingEmail": "john@example.com",
    "shippingAddressLine1": "123 Main St",
    "shippingAddressLine2": "",
    "shippingCity": "New York",
    "shippingState": "NY",
    "shippingPostalCode": "10001",
    "shippingCountry": "US",
    "shippingAddressType": "home",
    "method": "Card",
    "amount": 99.99,
    "shippingCost": 5.99,
    "shippingMethod": "Standard",
    "couponId": null,
    "couponCode": "",
    "discount": 0,
    "totalCost": 30.00,
    "grossProfit": 69.99,
    "status": "Pending",
    "stockDeducted": true,
    "paymentStatus": "pending",
    "fulfillmentStatus": "unfulfilled",
    "trackingNumber": "",
    "carrier": "",
    "shippedAt": null,
    "deliveredAt": null,
    "notes": "",
    "items": [
      {
        "id": 1,
        "productId": 2,
        "variantId": 5,
        "inventoryId": null,
        "productName": "T-Shirt",
        "variantName": "Small / Red",
        "quantity": 2,
        "unitPrice": 29.99,
        "totalPrice": 59.98,
        "unitCost": 15.00,
        "totalCost": 30.00
      }
    ],
    "shipments": [],
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**404:** `{"error": "Sell not found"}`

## GET `/sells/invoice/:invoiceNo` — 200 OK

Same response shape as `GET /sells/:id`.

**404:** `{"error": "Sell not found"}`

## POST `/sells/` — 201 Created

```json
{
  "message": "Sell created successfully",
  "data": { /* full sell object with customer, shipping address, and items preloaded */ }
}
```

## PATCH `/sells/:id/status` — 200 OK

```json
{
  "message": "Status updated successfully",
  "data": { /* sell object */ }
}
```

## DELETE `/sells/:id` — 200 OK

```json
{
  "message": "Sell deleted successfully"
}
```

## GET `/sells/stats` — 200 OK

```json
{
  "message": "Statistics retrieved successfully",
  "data": {
    "totalSells": 100,
    "totalRevenue": 9999.99,
    "totalCost": 6000.00,
    "grossProfit": 3999.99,
    "gpMarginPercent": 40.0,
    "pendingOrders": 10,
    "processingOrders": 5,
    "deliveredOrders": 85
  }
}
```

---

# Validation Rules

| Field          | Rule                                                                        |
|----------------|-----------------------------------------------------------------------------|
| `customerName` | Required                                                                    |
| `status`       | Must be one of: `Pending`, `Processing`, `Delivered`; default: `Pending`   |
| `method`       | Must be one of: `Cash`, `Card`, `Online`; default: `Cash`                   |
| `invoiceNo`    | Must be unique per company; auto-generated if empty                         |

**Inline shipping address validation (when `shippingFullName` + `shippingAddressLine1` provided):**
- `shippingFullName` — required
- `shippingAddressLine1` — required
- `shippingCity` — required
- `shippingCountry` — required
- `shippingAddressType` — defaults to `"other"` if not provided

**Saved address validation:**
- `shippingAddressId` must exist and belong to the provided `customerId`

---

# Business Logic Details

### Order Creation Transaction
The following steps are executed atomically inside a database transaction:
1. Parse `createSellInput`; resolve item `unitPrice` from `unitPrice` or `price` alias.
2. Auto-generate `invoice_no` (`INV-{unix_timestamp}`) if not provided.
3. Auto-set `order_time` to now if not provided.
4. Validate `customerName`, `status`, `method`.
5. Resolve shipping address via one of the three strategies.
6. Check `invoice_no` uniqueness within the company.
7. Create `sells` record and `order_items` records.
8. For each item: snapshot `cost_price` → `unit_cost`; compute `total_cost = unit_cost × quantity`.
9. Compute `sells.total_cost = SUM(order_items.total_cost)`.
10. Compute `sells.gross_profit = sells.amount - sells.total_cost`.
11. Deduct stock for each item (see Stock Deduction below).
12. Set `stock_deducted = true`.

### Stock Deduction
- **Variant items:**
  - Check `product_variants.stock >= qty`; error if insufficient.
  - If `inventory_id` is provided: deduct from that specific `variant_inventory` row.
  - Otherwise: deduct across available inventory rows, highest quantity first.
  - Sync `product_variants.stock = SUM(variant_inventory.quantity)` for the variant.
  - Sync `products.stock = SUM(variant stocks)` for the product.
- **Simple products:** Deduct from `products.stock` atomically.

### Order Deletion
- Soft-deletes the `sells` record.
- If `stock_deducted = true`, stock is restored (reverses the deduction process).
- Variant inventory rows are incremented; stocks are re-synced.

### Partial Update (PUT)
- Updates only the specified fields.
- Does **not** re-deduct or restore stock regardless of item quantity changes.
- Items are not updated via PUT; line items are immutable after creation through this endpoint.

---

# Dependencies

| Module      | Reason                                                                 |
|-------------|------------------------------------------------------------------------|
| `companies` | `company_id` scoping from JWT                                          |
| `customers` | `customer_id` FK; customer preloaded on fetch                          |
| `shipping`  | `shipping_address_id` FK; shipment records via `OrderShipments`        |
| `products`  | Stock deducted on create; restored on delete; cost price snapshotted   |
| `coupons`   | `coupon_id` FK; coupon code and discount amount recorded on order       |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` is extracted from the JWT and scopes all queries; users cannot access orders from another company.
- HTTP 401 is returned if `company_id` is not found in the JWT context.
- No additional role-based access control is documented at the endpoint level.

---

# Error Handling

| HTTP Status | Trigger                                                                                           |
|-------------|---------------------------------------------------------------------------------------------------|
| 400         | Invalid request body; missing `customerName`; invalid `status` or `method`; shipping address validation failure; insufficient stock for a variant; invalid shipping address ID; address does not belong to customer |
| 401         | `company_id` not found in JWT context                                                             |
| 404         | Sell not found by ID or invoice number                                                            |
| 409         | Duplicate `invoice_no` within the company                                                        |
| 500         | Database error; failed to create or delete sell; failed to restore stock                          |

**Error response examples:**
```json
{"error": "Customer name is required"}
{"error": "Invalid status. Must be one of: Pending, Processing, Delivered"}
{"error": "Invalid payment method. Must be one of: Cash, Card, Online"}
{"error": "Invoice number already exists"}
{"error": "insufficient stock for variant '...' (available: N, requested: N)"}
{"error": "Shipping address does not belong to this customer"}
```

---

# Testing Notes

- Verify `invoice_no` auto-generation and uniqueness enforcement (HTTP 409 on duplicate).
- Verify `customerName` is required (HTTP 400 if missing).
- Verify stock deduction is transactional: if stock deduction fails for any item, the entire order creation rolls back.
- Verify stock is restored on order deletion when `stock_deducted = true`.
- Verify all three shipping address resolution paths.
- Verify `unit_cost` is correctly snapshotted from the variant's `cost_price` when a variant is present, and from the product's `cost_price` for simple products.
- Verify `sells.total_cost` and `sells.gross_profit` are computed correctly from item totals.
- Verify `PUT /sells/:id` does not change stock values.
- Verify `PATCH /sells/:id/status` with invalid status returns HTTP 400.
- Verify `GET /sells/` filters: `status`, `method`, `customer_id`, `start_date`, `end_date`, and `limit` vs pagination behavior.
- Verify `GET /sells/stats` reflects only non-deleted orders.

---

# Open Questions / Missing Details

- Whether `PUT /sells/:id` can update line items (`order_items`) is not documented; the source states only certain top-level fields are updatable.
- Whether the `payment_status` and `fulfillment_status` fields can be updated through `PUT /sells/:id` is not specified.
- The exact date format accepted by `start_date` and `end_date` query parameters (`2006-01-02`) is Go's reference time format, but this should be validated by the caller.
- Whether `GET /sells/stats` is date-range filterable is not documented.
- The behavior of stock deduction for simple products when `inventory_id` is provided is not documented.

---

# Change Log / Notes

- Source documentation does not include a version history.
- This file was standardized from the original module notes on 2026-03-31.
