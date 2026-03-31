# Module: Vendor Returns

## Business Purpose
Manages product returns sent back to vendors/suppliers. When a return is created, inventory is immediately deducted (items are being shipped back). Tracks return status lifecycle and credit type received from vendor.

---

## Database Tables

### Table: `vendor_returns`
| Column         | Type          | Notes                                                             |
|----------------|---------------|-------------------------------------------------------------------|
| id             | uint (PK, AI) |                                                                   |
| company_id     | uint (FK)     | FK → companies.id; indexed; json: companyId                       |
| return_number  | varchar(100)  | unique; not null; json: returnNumber                              |
| vendor_id      | uint (FK)     | FK → vendors.id; not null; json: vendorId                         |
| vendor_name    | varchar(255)  | not null; json: vendorName                                        |
| total_amount   | float64       | not null; default 0; json: totalAmount                            |
| status         | string        | 'pending','shipped','received_by_vendor','completed'; default 'pending' |
| return_date    | timestamp     | not null; ordered DESC by default; json: returnDate               |
| completed_date | *timestamp    | nullable; set when status becomes 'completed'; json: completedDate|
| credit_type    | string        | not null; one of: refund, credit_note, replacement; json: creditType |
| notes          | text          | json: notes                                                       |
| created_by     | string        | json: createdBy                                                   |
| created_at     | timestamp     | json: createdAt                                                   |
| updated_at     | timestamp     | json: updatedAt                                                   |
| deleted_at     | timestamp     | Soft delete (json: "-")                                           |

### Table: `vendor_return_items`
| Column       | Type          | Notes                              |
|--------------|---------------|------------------------------------|
| id           | uint (PK, AI) |                                    |
| return_id    | uint (FK)     | FK → vendor_returns.id; not null   |
| product_id   | *uint (FK)    | FK → products.id; nullable         |
| product_name | string        | not null; json: productName        |
| variant_id   | *uint (FK)    | FK → product_variants.id; nullable |
| variant_name | string        | json: variantName                  |
| quantity     | int           | not null; default 1                |
| unit_price   | float64       | not null; default 0; json: unitPrice|
| total_price  | float64       | not null; default 0; json: totalPrice|
| reason       | string        | not null; json: reason             |
| created_at   | timestamp     |                                    |
| updated_at   | timestamp     |                                    |

---

## Relationships
- VendorReturn → Vendor (BelongsTo)
- VendorReturn → VendorReturnItems (HasMany via return_id)
- VendorReturnItem → Product (BelongsTo, optional)
- VendorReturnItem → ProductVariant (BelongsTo, optional)

---

## Business Logic
- **Return number**: auto-generated as `VRT-{unix_microseconds}` if not provided.
- **Return date**: set to now if not provided.
- **Default status**: `"pending"` if not provided.
- **Credit type validation**: must be one of: `refund`, `credit_note`, `replacement`.
- **Status validation**: must be one of: `pending`, `shipped`, `received_by_vendor`, `completed`.
- **Inventory deduction on create**: each item deducts from inventory in a transaction:
  - For variant items: deduct from `variant_inventory` (skip if no inventory record found)
  - For simple products: deduct from `products.stock` (error if insufficient)
  - Returns `400` if insufficient stock or inventory
- **completed_date**: automatically set when status becomes `"completed"` (on both UpdateVendorReturn and UpdateVendorReturnStatus).
- Scoped by `company_id` from JWT context.

---

## Endpoints

### GET /vendor-returns/
**Auth:** Bearer JWT required

**Query Parameters:**
| Param     | Type   | Default | Description                                                   |
|-----------|--------|---------|---------------------------------------------------------------|
| page      | int    | 1       |                                                               |
| per_page  | int    | 10      |                                                               |
| search    | string |         | LIKE match on return_number, vendor_name                      |
| status    | string |         | pending/shipped/received_by_vendor/completed; "all" = no filter |
| vendor_id | uint   |         | Filter by vendor                                              |

**Response 200:**
```json
{
  "message": "Vendor returns retrieved successfully",
  "data": [ /* array of return objects with vendor and items */ ],
  "pagination": { "page": 1, "per_page": 10, "total": 8 }
}
```

---

### GET /vendor-returns/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Vendor return fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "returnNumber": "VRT-1700000000000",
    "vendorId": 2,
    "vendor": { ... },
    "vendorName": "Acme Supplies",
    "totalAmount": 299.99,
    "status": "pending",
    "returnDate": "2024-01-10T00:00:00Z",
    "completedDate": null,
    "creditType": "refund",
    "notes": "",
    "createdBy": "",
    "items": [
      {
        "id": 1,
        "returnId": 1,
        "productId": 2,
        "productName": "T-Shirt",
        "variantId": 5,
        "variantName": "Small / Red",
        "quantity": 10,
        "unitPrice": 15.00,
        "totalPrice": 150.00,
        "reason": "Defective batch"
      }
    ],
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**Error:** `404` — `{"error": "Vendor return not found"}`

---

### POST /vendor-returns/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "returnNumber": "",
  "vendorId": 2,
  "vendorName": "Acme Supplies",
  "totalAmount": 299.99,
  "status": "pending",
  "returnDate": "2024-01-10T00:00:00Z",
  "creditType": "refund",
  "notes": "",
  "items": [
    {
      "productId": 2,
      "productName": "T-Shirt",
      "variantId": 5,
      "variantName": "Small / Red",
      "quantity": 10,
      "unitPrice": 15.00,
      "totalPrice": 150.00,
      "reason": "Defective batch"
    }
  ]
}
```

**Validation:**
- `vendorName` — required
- `creditType` — required; one of: refund, credit_note, replacement
- `status` — one of: pending, shipped, received_by_vendor, completed (default: pending)

**Response 201:**
```json
{
  "message": "Vendor return created and inventory deducted successfully",
  "data": { /* return object with vendor and items */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `400` — `{"error": "Vendor name is required"}`
- `400` — `{"error": "Invalid status"}`
- `400` — `{"error": "Invalid credit type"}`
- `400` — `{"error": "insufficient inventory for variant ..."}`
- `400` — `{"error": "insufficient stock for product ..."}`
- `409` — `{"error": "Return number already exists"}`
- `500` — `{"error": "Failed to create vendor return"}`

---

### PUT /vendor-returns/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Update status and/or notes. Sets completed_date when status → "completed".

**Request Body:**
```json
{
  "status": "shipped",
  "notes": "Shipped via DHL"
}
```

**Response 200:**
```json
{
  "message": "Vendor return updated successfully",
  "data": { /* return object with vendor and items */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body"}`
- `400` — `{"error": "Invalid status"}`
- `404` — `{"error": "Vendor return not found"}`
- `500` — `{"error": "Failed to update vendor return"}`

---

### PATCH /vendor-returns/:id/status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "status": "completed"
}
```

**Validation:** `status` — required; one of valid statuses

**Response 200:**
```json
{
  "message": "Status updated successfully",
  "data": { /* return object with vendor and items */ }
}
```

**Error Responses:**
- `400` — `{"error": "Status is required"}` / `{"error": "Invalid status"}`
- `404` — `{"error": "Vendor return not found"}`
- `500` — `{"error": "Failed to update status"}`

---

### DELETE /vendor-returns/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Vendor return deleted successfully"
}
```

**Error:** `404` — `{"error": "Vendor return not found"}`

---

### GET /vendor-returns/stats
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Statistics retrieved successfully",
  "data": {
    "total": 20,
    "pending": 5,
    "shipped": 8,
    "completed": 7,
    "totalCreditAmount": 5999.99
  }
}
```

---

### GET /vendor-returns/vendor/:vendorId
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Vendor returns retrieved successfully",
  "data": [ /* array of return objects for vendor, ordered by returnDate DESC */ ]
}
```

---

## Dependencies
- **vendors** — returns reference vendors
- **products** — inventory is deducted on create
