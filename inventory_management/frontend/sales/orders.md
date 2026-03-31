# Frontend Module: Orders (Sells)

## Pages
- `/dashboard/sales/orders` — Order list with filters and stats
- `/dashboard/sales/orders/new` — Create new order (POS-style)
- `/dashboard/sales/orders/:id` — Order detail / edit view
- `/dashboard/sales/orders/invoice/:invoiceNo` — View order by invoice number

## API Service
`lib/sellsApi.ts`

---

## GET /sells
**Purpose:** Fetch paginated list of orders with optional filters
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by invoice number, customer name (optional)
- `status` — filter by status: `"Pending"` | `"Processing"` | `"Delivered"` (optional)
- `method` — filter by payment method (optional)
- `start_date` — filter from date (optional)
- `end_date` — filter to date (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "invoiceNo": "INV-20240101-001",
      "invoice_no": "INV-20240101-001",
      "orderTime": "2024-01-01T10:00:00Z",
      "customerId": 5,
      "customerName": "John Doe",
      "shippingAddressId": 3,
      "shippingAddress": {
        "id": 3,
        "customerId": 5,
        "fullName": "John Doe",
        "phone": "+1234567890",
        "email": "john@example.com",
        "addressLine1": "123 Main St",
        "addressLine2": "Apt 4B",
        "city": "New York",
        "state": "NY",
        "postalCode": "10001",
        "country": "US",
        "isDefault": true,
        "addressType": "shipping",
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-01T00:00:00Z"
      },
      "shippingFullName": "John Doe",
      "shippingPhone": "+1234567890",
      "shippingEmail": "john@example.com",
      "shippingAddressLine1": "123 Main St",
      "shippingAddressLine2": "Apt 4B",
      "shippingCity": "New York",
      "shippingState": "NY",
      "shippingPostalCode": "10001",
      "shippingCountry": "US",
      "shippingAddressType": "shipping",
      "method": "cash | card | online",
      "amount": 99.99,
      "discount": 10.00,
      "couponId": 2,
      "couponCode": "SAVE10",
      "shippingCost": 5.00,
      "status": "Pending | Processing | Delivered",
      "paymentStatus": "string",
      "fulfillmentStatus": "string",
      "trackingNumber": "string",
      "carrier": "string",
      "shippedAt": "2024-01-02T00:00:00Z",
      "notes": "string",
      "items": [
        {
          "id": 1,
          "sellId": 1,
          "productId": 10,
          "variantId": 2,
          "variant_id": 2,
          "inventoryId": 5,
          "inventory_id": 5,
          "productName": "Blue T-Shirt",
          "quantity": 2,
          "price": 29.99,
          "unit_price": 29.99,
          "unitPrice": 29.99,
          "total_price": 59.98,
          "totalPrice": 59.98,
          "createdAt": "2024-01-01T00:00:00Z",
          "updatedAt": "2024-01-01T00:00:00Z"
        }
      ],
      "shipments": [
        {
          "id": 1,
          "sellId": 1,
          "trackingNumber": "1Z999AA1",
          "carrier": "UPS",
          "shippingMethod": "ground",
          "status": "in_transit",
          "estimatedDelivery": "2024-01-05T00:00:00Z",
          "shippingCost": 5.00,
          "weight": 1.5,
          "dimensions": "10x8x4",
          "notes": "string",
          "createdAt": "2024-01-01T00:00:00Z",
          "updatedAt": "2024-01-01T00:00:00Z"
        }
      ],
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 250,
  "page": 1,
  "limit": 20
}
```

**Note:** Both `invoiceNo` and `invoice_no` may be present (camelCase and snake_case). Similarly, `SellItem` has both `variantId`/`variant_id`, `inventoryId`/`inventory_id`, `unitPrice`/`unit_price`, `totalPrice`/`total_price` — the backend appears to return both forms; frontend uses whichever is non-null.

**Frontend Impact:**
- Order list table with status filter tabs (Pending/Processing/Delivered)
- Date range picker drives `start_date`/`end_date`
- Payment method filter dropdown drives `method`

---

## GET /sells/stats
**Purpose:** Fetch aggregate order statistics for the dashboard
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "total_sells": 250,
    "total_revenue": 18750.50,
    "pending_count": 15,
    "processing_count": 8,
    "delivered_count": 227
  }
}
```

**Frontend Impact:**
- Stats cards at top of orders page: total orders, total revenue, pending count
- Used for dashboard summary widgets

---

## GET /sells/:id
**Purpose:** Fetch full order details including items, shipments, and shipping address
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric order ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full SellResponse object (same shape as list items above)..." }
}
```

**Frontend Impact:**
- Order detail page shows line items, totals, shipment tracking, shipping address
- Edit order form pre-populated

---

## GET /sells/invoice/:invoiceNo
**Purpose:** Fetch order by invoice number (URL-encoded)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `invoiceNo` — invoice number string, URL-encoded before sending

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full SellResponse object..." }
}
```

**Frontend Impact:**
- Invoice search/lookup feature
- Print invoice page loaded by invoice number from URL

---

## POST /sells
**Purpose:** Create a new order
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "customerId": 5,
  "customerName": "John Doe",
  "customerEmail": "john@example.com",
  "customerPhone": "+1234567890",
  "shippingAddressId": 3,
  "method": "cash | card | online",
  "amount": 99.99,
  "discount": 10.00,
  "couponId": 2,
  "couponCode": "SAVE10",
  "shippingCost": 5.00,
  "status": "Pending | Processing | Delivered",
  "note": "string",
  "items": [
    {
      "productId": 10,
      "variantId": 2,
      "variant_id": 2,
      "inventoryId": 5,
      "inventory_id": 5,
      "productName": "Blue T-Shirt",
      "quantity": 2,
      "price": 29.99
    }
  ]
}
```

**Note:** `customerName` and `items` array are required. `customerId` is optional (allows guest orders). `items` must contain at least one item. Both `variantId`/`variant_id` and `inventoryId`/`inventory_id` are duplicated — send both for compatibility.

**Expected Response 201:**
```json
{
  "message": "Order created successfully",
  "data": { "...full SellResponse object with generated invoiceNo..." }
}
```

**Frontend Impact:**
- On success: navigate to order detail or print invoice
- Stock quantity decremented (backend-side)
- Customer order history updated

---

## PUT /sells/:id
**Purpose:** Update order details (not for status changes — use PATCH for status)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric order ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "customerName": "string",
  "customerEmail": "string",
  "customerPhone": "string",
  "method": "string",
  "amount": 99.99,
  "discount": 10.00,
  "shippingCost": 5.00,
  "status": "Pending | Processing | Delivered",
  "notes": "string"
}
```

**Note:** All fields optional. `notes` key used here (not `note` as in create). The `items` array is NOT updatable via this endpoint.

**Expected Response 200:**
```json
{
  "message": "Order updated successfully",
  "data": { "...full SellResponse object..." }
}
```

**Frontend Impact:**
- Edit order form for non-status fields
- Items cannot be changed after creation (backend constraint)

---

## PATCH /sells/:id/status
**Purpose:** Update only the order status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric order ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "status": "Pending | Processing | Delivered"
}
```

**Expected Response 200:**
```json
{
  "message": "Order status updated",
  "data": { "...full SellResponse object..." }
}
```

**Frontend Impact:**
- Status dropdown in order list row (inline update)
- Status badge on order detail page
- Status change to "Delivered" may trigger shipment status update (backend-dependent)

---

## DELETE /sells/:id
**Purpose:** Delete an order
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric order ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Order deleted successfully"
}
```

**Frontend Impact:**
- Order removed from list
- Stock may or may not be restored (backend-dependent behavior)
- Should only be allowed for Pending orders (client-side guard recommended)
