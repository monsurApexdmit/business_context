# Module: Orders (Sells)

## Business Purpose
Manages sales orders. An order (called "Sell" internally) captures the customer, items purchased, shipping information, payment method, and financial totals. Stock is deducted transactionally on order creation and restored on deletion. Supports inline or saved shipping addresses. Orders are scoped by company_id.

---

## Database Tables

### Table: `sells`
| Column                | Type           | Notes                                                                 |
|-----------------------|----------------|-----------------------------------------------------------------------|
| id                    | uint (PK, AI)  | json: id                                                              |
| company_id            | uint           | not null; indexed; json: companyId                                    |
| invoice_no            | varchar(100)   | unique; not null; json: invoiceNo                                     |
| order_time            | timestamp      | not null; json: orderTime                                             |
| customer_id           | *uint (FK)     | FK → customers.id; nullable; json: customerId                         |
| customer_name         | varchar(255)   | not null; json: customerName                                          |
| shipping_address_id   | *uint (FK)     | FK → shipping_addresses.id; nullable; json: shippingAddressId         |
| shipping_full_name    | varchar(255)   | snapshot; json: shippingFullName                                      |
| shipping_phone        | varchar(50)    | snapshot; json: shippingPhone                                         |
| shipping_email        | varchar(255)   | snapshot; json: shippingEmail                                         |
| shipping_address_line1| varchar(255)   | snapshot; json: shippingAddressLine1                                  |
| shipping_address_line2| varchar(255)   | snapshot; json: shippingAddressLine2                                  |
| shipping_city         | varchar(100)   | snapshot; json: shippingCity                                          |
| shipping_state        | varchar(100)   | snapshot; json: shippingState                                         |
| shipping_postal_code  | varchar(20)    | snapshot; json: shippingPostalCode                                    |
| shipping_country      | varchar(100)   | snapshot; json: shippingCountry                                       |
| shipping_address_type | varchar(50)    | snapshot; json: shippingAddressType                                   |
| method                | varchar(100)   | 'Cash', 'Card', 'Online'; default 'Cash'; json: method                |
| amount                | float64        | default 0; json: amount                                               |
| shipping_cost         | float64        | default 0; json: shippingCost                                         |
| shipping_method       | varchar(100)   | json: shippingMethod                                                  |
| coupon_id             | *uint (FK)     | FK → coupons.id; nullable; indexed; json: couponId                    |
| coupon_code           | string         | json: couponCode                                                      |
| discount              | float64        | default 0; json: discount                                             |
| status                | varchar(50)    | 'Pending', 'Processing', 'Delivered'; default 'Pending'; json: status |
| stock_deducted        | bool           | not null; default false; json: stockDeducted                          |
| payment_status        | enum           | 'pending','paid','partially_paid','refunded','failed'; default 'pending'; json: paymentStatus |
| fulfillment_status    | enum           | 'unfulfilled','processing','shipped','delivered','cancelled'; default 'unfulfilled'; json: fulfillmentStatus |
| tracking_number       | varchar(100)   | json: trackingNumber                                                  |
| carrier               | varchar(100)   | json: carrier                                                         |
| shipped_at            | *timestamp     | nullable; json: shippedAt                                             |
| delivered_at          | *timestamp     | nullable; json: deliveredAt                                           |
| total_cost            | float64        | default 0; SUM of order_items.total_cost; json: totalCost             |
| gross_profit          | float64        | default 0; amount - total_cost; json: grossProfit                     |
| notes                 | text           | json: notes                                                           |
| created_at            | timestamp      | json: createdAt                                                       |
| updated_at            | timestamp      | json: updatedAt                                                       |
| deleted_at            | timestamp      | Soft delete (json: "-")                                               |

### Table: `order_items`
| Column       | Type          | Notes                                                          |
|--------------|---------------|----------------------------------------------------------------|
| id           | uint (PK, AI) |                                                                |
| sell_id      | uint (FK)     | FK → sells.id                                                  |
| product_id   | *uint (FK)    | FK → products.id; nullable                                     |
| variant_id   | *uint (FK)    | FK → product_variants.id; nullable                             |
| inventory_id | *uint (FK)    | FK → variant_inventory.id; nullable                            |
| product_name | string        | json: productName                                              |
| variant_name | string        | json: variantName                                              |
| quantity     | int           | json: quantity                                                 |
| unit_price   | float64       | json: unitPrice                                                |
| total_price  | float64       | json: totalPrice                                               |
| unit_cost    | float64       | default 0; cost_price snapshotted from product/variant at sale time; json: unitCost |
| total_cost   | float64       | default 0; unit_cost × quantity; json: totalCost               |

---

## Relationships
- Sell → Customer (BelongsTo, optional)
- Sell → ShippingAddress (BelongsTo, optional)
- Sell → Coupon (BelongsTo, optional)
- Sell → OrderItems (HasMany via sell_id)
- Sell → OrderShipments (HasMany via sell_id)

---

## Business Logic

### Order Creation
1. Parse `createSellInput` (supports both camelCase field names)
2. Resolve `unitPrice` from either `unitPrice` or `price` field on each item
3. Auto-generate invoice number if not provided (`INV-{unix_timestamp}`)
4. Auto-set order_time to now if not provided
5. Validate customerName (required), status, method
6. Shipping address resolution (three options):
   - **Option 1**: inline custom address (shippingFullName + shippingAddressLine1 provided) — validate required fields, default type to "other"
   - **Option 2**: saved address by `shippingAddressId` — validate ownership
   - **Option 3**: auto-select customer's default address if customer_id provided
7. Check invoice number uniqueness within company
8. Execute in DB transaction:
   - Create sell + items
   - For each item: snapshot `cost_price` from variant (if variant exists) or product into `order_items.unit_cost`; compute `order_items.total_cost = unit_cost × quantity`
   - Compute `sells.total_cost = SUM(order_items.total_cost)`
   - Compute `sells.gross_profit = sells.amount - sells.total_cost`
   - Deduct stock for each item
   - Mark `stock_deducted = true`

### Stock Deduction (deductOrderItemsStock)
- For variant items:
  - Check `product_variants.stock >= qty`; error if insufficient
  - If `inventory_id` provided: deduct from that specific `variant_inventory` row
  - Otherwise: deduct across available inventory rows from highest quantity first
  - Sync `product_variants.stock` = SUM of `variant_inventory.quantity`
  - Sync `products.stock` = SUM of variant stocks
- For simple products: deduct from `products.stock` atomically

### Order Deletion
- Restores stock if `stock_deducted = true`
- Stock restoration reverses the deduction process (adds back to variant_inventory, then syncs)

### Invoice Number Format
Auto-generated: `INV-{unix_timestamp}` (e.g. `INV-1700000000`)

---

## Endpoints

### GET /sells/
**Auth:** Bearer JWT required

**Query Parameters:**
| Param       | Type   | Default | Description                                    |
|-------------|--------|---------|------------------------------------------------|
| page        | int    | 1       | Pagination page (ignored if `limit` is set)    |
| per_page    | int    | 10      | Items per page                                 |
| limit       | int    |         | If set (not "all"), returns exactly N items with no pagination metadata |
| search      | string |         | LIKE match on customer_name                    |
| status      | string |         | Pending / Processing / Delivered; "all" = no filter |
| method      | string |         | Cash / Card / Online; "all" = no filter        |
| customer_id | uint   |         | Filter by customer                             |
| start_date  | string |         | Filter by order_time >= (format: 2006-01-02)   |
| end_date    | string |         | Filter by order_time <= end of that day        |

**Response 200 (paginated):**
```json
{
  "message": "Sells retrieved successfully",
  "data": [ /* array of sell objects */ ],
  "pagination": {
    "page": 1,
    "per_page": 10,
    "total": 100
  }
}
```

**Response 200 (with limit param, no pagination):**
```json
{
  "message": "Sells retrieved successfully",
  "data": [ /* array of sell objects */ ]
}
```

---

### GET /sells/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Sell fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "invoiceNo": "INV-1700000000",
    "orderTime": "2024-01-01T10:00:00Z",
    "customerId": 5,
    "customer": { ... },
    "customerName": "John Doe",
    "shippingAddressId": 3,
    "shippingAddress": { ... },
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

**Error:** `404` — `{"error": "Sell not found"}`

---

### GET /sells/invoice/:invoiceNo
**Auth:** Bearer JWT required

**Response 200:** Same as GET /sells/:id
**Error:** `404` — `{"error": "Sell not found"}`

---

### POST /sells/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
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

**Validation:**
- `customerName` — required
- `status` — must be: Pending, Processing, Delivered (default: Pending)
- `method` — must be: Cash, Card, Online (default: Cash)
- `invoiceNo` — auto-generated if empty; must be unique per company

**Response 201:**
```json
{
  "message": "Sell created successfully",
  "data": { /* sell object with customer, shipping address, items preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `400` — `{"error": "Customer name is required"}`
- `400` — `{"error": "Invalid status. Must be one of: Pending, Processing, Delivered"}`
- `400` — `{"error": "Invalid payment method. Must be one of: Cash, Card, Online"}`
- `400` — `{"error": "Shipping full name is required when providing custom address"}`
- `400` — `{"error": "Shipping address line 1 is required when providing custom address"}`
- `400` — `{"error": "Shipping city is required when providing custom address"}`
- `400` — `{"error": "Shipping country is required when providing custom address"}`
- `400` — `{"error": "Invalid shipping address ID"}`
- `400` — `{"error": "Shipping address does not belong to this customer"}`
- `400` — `{"error": "insufficient stock for variant '...' (available: N, requested: N)"}`
- `401` — `{"error": "Company ID not found in context"}`
- `409` — `{"error": "Invoice number already exists"}`
- `500` — `{"error": "Failed to create sell"}`

---

### PUT /sells/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Partial update of sell fields (does NOT re-deduct/restore stock).

**Request Body:** Any subset of sell fields (Sell model structure).

**Updatable Fields:** invoiceNo, orderTime, customerId, customerName, method, amount, shippingCost, discount, status, notes

**Response 200:**
```json
{
  "message": "Sell updated successfully",
  "data": { /* sell object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body"}`
- `400` — `{"error": "Invalid payment method"}`
- `400` — `{"error": "Invalid status"}`
- `404` — `{"error": "Sell not found"}`
- `500` — `{"error": "Failed to update sell"}`

---

### PATCH /sells/:id/status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "status": "Processing"
}
```

**Validation:** `status` — required; must be: Pending, Processing, Delivered

**Response 200:**
```json
{
  "message": "Status updated successfully",
  "data": { /* sell object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Status is required"}`
- `400` — `{"error": "Invalid status. Must be one of: Pending, Processing, Delivered"}`
- `404` — `{"error": "Sell not found"}`
- `500` — `{"error": "Failed to update status"}`

---

### DELETE /sells/:id
**Auth:** Bearer JWT required
**Purpose:** Soft-delete the order and restore stock if it was deducted.

**Response 200:**
```json
{
  "message": "Sell deleted successfully"
}
```

**Error Responses:**
- `404` — `{"error": "Sell not found"}`
- `500` — `{"error": "Failed to delete sell and restore stock"}`

---

### GET /sells/stats
**Auth:** Bearer JWT required

**Response 200:**
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

## Dependencies
- **customers** — sell.customer_id references customers
- **shipping** — shipping addresses and shipment records
- **products** — stock deducted on order create; restored on delete
- **coupons** — sell may reference a coupon
