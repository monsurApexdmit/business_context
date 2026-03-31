# Module: Customer Returns

## Business Purpose
Manages return requests from customers. A return can contain multiple items. Returns go through a status lifecycle (pending → approved/rejected → completed). On approval, inventory is restocked. On creation, product and variant names are auto-filled from DB if not provided.

---

## Database Tables

### Table: `customer_returns`
| Column         | Type          | Notes                                                         |
|----------------|---------------|---------------------------------------------------------------|
| id             | uint (PK, AI) |                                                               |
| company_id     | uint (FK)     | FK → companies.id; indexed; json: companyId                   |
| return_number  | varchar(100)  | unique; not null; json: returnNumber                          |
| customer_id    | *uint (FK)    | FK → customers.id; nullable; json: customerId                 |
| customer_name  | varchar(255)  | json: customerName                                            |
| order_id       | *uint (FK)    | FK → sells.id; nullable; json: orderId                        |
| order_number   | varchar(100)  | json: orderNumber                                             |
| total_amount   | float64       | not null; default 0; json: totalAmount                        |
| status         | varchar(50)   | 'pending','approved','rejected','completed'; default 'pending' |
| request_date   | timestamp     | not null; json: requestDate; ordered DESC by default          |
| processed_date | *timestamp    | nullable; json: processedDate                                 |
| refund_method  | varchar(100)  | not null; one of: cash, store_credit, original_payment; json: refundMethod |
| notes          | text          | json: notes                                                   |
| processed_by   | varchar(255)  | json: processedBy                                             |
| created_at     | timestamp     | json: createdAt                                               |
| updated_at     | timestamp     | json: updatedAt                                               |
| deleted_at     | timestamp     | Soft delete (json: "-")                                       |

### Table: `customer_return_items`
| Column       | Type          | Notes                              |
|--------------|---------------|------------------------------------|
| id           | uint (PK, AI) |                                    |
| return_id    | uint (FK)     | FK → customer_returns.id; not null |
| product_id   | *uint (FK)    | FK → products.id; nullable         |
| product_name | varchar(255)  | not null; json: productName        |
| variant_id   | *uint (FK)    | FK → product_variants.id; nullable |
| variant_name | varchar(255)  | json: variantName                  |
| quantity     | int           | not null; default 1                |
| price        | float64       | not null; default 0                |
| reason       | varchar(255)  | not null; json: reason             |
| created_at   | timestamp     |                                    |
| updated_at   | timestamp     |                                    |

---

## Relationships
- CustomerReturn → Customer (BelongsTo, optional)
- CustomerReturn → Sell/Order (BelongsTo via order_id, optional)
- CustomerReturn → CustomerReturnItems (HasMany via return_id)
- CustomerReturnItem → Product (BelongsTo, optional)
- CustomerReturnItem → ProductVariant (BelongsTo, optional)

---

## Business Logic
- **Return number**: auto-generated as `RET-{unix_microseconds}` if not provided.
- **Request date**: set to now if not provided.
- **Default status**: `"pending"` if not provided.
- **Refund method validation**: must be one of: `cash`, `store_credit`, `original_payment`.
- **Product/variant names**: auto-filled from DB if items have ID but no name.
- **ApproveReturn**: only `pending` returns can be approved.
  - For variant items: restocks `variant_inventory` (creates row at location_id=1 if none exists)
  - For simple products: increments `products.stock`
  - Sets `processed_date` to now, `status` to `"approved"`
- **RejectReturn**: only `pending` returns can be rejected. Sets status to `"rejected"` and `processed_date` to now.
- Scoped by `company_id` from JWT context.

---

## Endpoints

### GET /customer-returns/
**Auth:** Bearer JWT required

**Query Parameters:**
| Param       | Type   | Default | Description                                                       |
|-------------|--------|---------|-------------------------------------------------------------------|
| page        | int    | 1       |                                                                   |
| per_page    | int    | 10      |                                                                   |
| search      | string |         | LIKE match on return_number, customer_name, order_number          |
| status      | string |         | pending/approved/rejected/completed; "all" = no filter            |
| customer_id | uint   |         | Filter by customer                                                |

**Response 200:**
```json
{
  "message": "Customer returns retrieved successfully",
  "data": [ /* array of return objects with customer and items */ ],
  "pagination": { "page": 1, "per_page": 10, "total": 15 }
}
```

---

### GET /customer-returns/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Customer return fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "returnNumber": "RET-1700000000000",
    "customerId": 5,
    "customer": { ... },
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

**Error:** `404` — `{"error": "Customer return not found"}`

---

### POST /customer-returns/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
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

**Validation:**
- `refundMethod` — required; one of: cash, store_credit, original_payment
- `status` — one of: pending, approved, rejected, completed (default: pending)

**Response 201:**
```json
{
  "message": "Customer return created successfully",
  "data": { /* return object with customer and items */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `400` — `{"error": "Invalid status"}`
- `400` — `{"error": "Invalid refund method"}`
- `409` — `{"error": "Return number already exists"}`
- `500` — `{"error": "Failed to create customer return"}`

---

### PUT /customer-returns/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Update status, notes, or processed_by.

**Request Body:**
```json
{
  "status": "completed",
  "notes": "Refund processed",
  "processedBy": "Admin"
}
```

**Response 200:**
```json
{
  "message": "Customer return updated successfully",
  "data": { /* return object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid status"}`
- `404` — `{"error": "Customer return not found"}`
- `500` — `{"error": "Failed to update customer return"}`

---

### POST /customer-returns/:id/approve
**Auth:** Bearer JWT required
**Purpose:** Approve a pending return and restock inventory.

**Response 200:**
```json
{
  "message": "Customer return approved and inventory restocked successfully",
  "data": { /* return object with status: "approved" */ }
}
```

**Error Responses:**
- `400` — `{"error": "Can only approve pending returns"}`
- `404` — `{"error": "Customer return not found"}`
- `500` — `{"error": "Failed to approve return and restock inventory"}`

---

### POST /customer-returns/:id/reject
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (optional):**
```json
{
  "notes": "Return request rejected due to policy"
}
```

**Response 200:**
```json
{
  "message": "Customer return rejected successfully",
  "data": { /* return object with status: "rejected" */ }
}
```

**Error Responses:**
- `400` — `{"error": "Can only reject pending returns"}`
- `404` — `{"error": "Customer return not found"}`
- `500` — `{"error": "Failed to reject return"}`

---

### DELETE /customer-returns/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Customer return deleted successfully"
}
```

**Error:** `404` — `{"error": "Customer return not found"}`

---

### GET /customer-returns/stats
**Auth:** Bearer JWT required

**Response 200:**
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

---

### GET /customer-returns/customer/:customerId
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Customer returns retrieved successfully",
  "data": [ /* array of return objects for customer, ordered by requestDate DESC */ ]
}
```

---

## Dependencies
- **customers** — returns reference customers
- **orders** — returns can reference original orders
- **products** — inventory is restocked on approval
