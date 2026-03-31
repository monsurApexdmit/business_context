# Frontend Module: Vendor Returns

## Pages
- `/dashboard/procurement/vendor-returns` — Vendor return list with status filters
- `/dashboard/procurement/vendor-returns/new` — Create vendor return form
- `/dashboard/procurement/vendor-returns/:id` — Vendor return detail with status management

## API Service
`lib/vendorReturnsApi.ts`

---

## GET /vendor-returns
**Purpose:** Fetch paginated list of vendor return requests
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by return number or vendor name (optional)
- `status` — filter by status: `"pending"` | `"shipped"` | `"received_by_vendor"` | `"completed"` (optional)
- `vendorId` — filter returns for a specific vendor (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "returnNumber": "VRT-20240101-001",
      "vendorId": 3,
      "vendorName": "Acme Supplies",
      "status": "pending | shipped | received_by_vendor | completed",
      "creditType": "refund | replacement | store_credit",
      "totalAmount": 250.00,
      "notes": "string",
      "createdBy": "string",
      "returnDate": "2024-01-05T00:00:00Z",
      "completedDate": null,
      "items": [
        {
          "id": 1,
          "vendorReturnId": 1,
          "productId": 10,
          "productName": "Widget A",
          "variantId": null,
          "variantName": null,
          "quantity": 5,
          "unitPrice": 50.00,
          "totalPrice": 250.00,
          "reason": "Damaged goods received"
        }
      ],
      "createdAt": "2024-01-04T00:00:00Z",
      "updatedAt": "2024-01-05T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 15
  },
  "total": 15
}
```

**Note:** Pagination uses `per_page` in the response object. Both `pagination.total` and top-level `total` may be present.

**Frontend Impact:**
- Vendor returns list with status tabs (Pending / Shipped / Received / Completed)
- `totalAmount` shown as financial impact column

---

## GET /vendor-returns/stats
**Purpose:** Fetch aggregate vendor return statistics
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "total": 15,
    "pending": 4,
    "shipped": 3,
    "received_by_vendor": 2,
    "completed": 6,
    "total_credit_amount": 1800.00,
    "totalCreditAmount": 1800.00
  }
}
```

**Note:** Both `total_credit_amount` (snake_case) and `totalCreditAmount` (camelCase) may be present — frontend should handle both.

**Frontend Impact:**
- Stats cards at top of vendor returns page
- Total credit amount in procurement financial summary

---

## GET /vendor-returns/:id
**Purpose:** Fetch a single vendor return with full item details
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric vendor return ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full VendorReturnResponse object..." }
}
```

**Frontend Impact:**
- Return detail page with items, amounts, and status management
- Status update dropdown visible based on current status

---

## GET /vendor-returns/vendor/:vendorId
**Purpose:** Fetch all returns associated with a specific vendor
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `vendorId` — numeric vendor ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [ "...array of VendorReturnResponse objects..." ],
  "pagination": { "page": 1, "per_page": 20, "total": 5 },
  "total": 5
}
```

**Frontend Impact:**
- Vendor profile page "Returns" tab
- Financial reconciliation per vendor

---

## POST /vendor-returns
**Purpose:** Create a new vendor return request
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "vendorId": 3,
  "vendorName": "Acme Supplies",
  "creditType": "refund",
  "notes": "string",
  "returnDate": "2024-01-05T00:00:00Z",
  "totalAmount": 250.00,
  "items": [
    {
      "productId": 10,
      "productName": "Widget A",
      "variantId": null,
      "quantity": 5,
      "unitPrice": 50.00,
      "totalPrice": 250.00,
      "reason": "Damaged goods received"
    }
  ]
}
```

**Note:** `vendorId` is required. `items` array is required with at least one item. Each item requires `productId`, `quantity`, and `reason`. `vendorName` optional (for display purposes). `totalAmount` optional (may be computed by backend from items).

**Expected Response 201:**
```json
{
  "message": "Vendor return created successfully",
  "data": { "...full VendorReturnResponse with returnNumber and status: 'pending'..." }
}
```

**Frontend Impact:**
- New vendor return created with `status: "pending"` automatically
- Return number generated by backend

---

## PUT /vendor-returns/:id
**Purpose:** Update a vendor return (before completion)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric vendor return ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "vendorId": 3,
  "creditType": "replacement",
  "notes": "Updated notes",
  "returnDate": "2024-01-06T00:00:00Z",
  "totalAmount": 300.00,
  "items": [
    {
      "productId": 10,
      "quantity": 6,
      "unitPrice": 50.00,
      "totalPrice": 300.00,
      "reason": "Corrected quantity"
    }
  ]
}
```

**Note:** All fields optional (`Partial<CreateVendorReturnData>`).

**Expected Response 200:**
```json
{
  "message": "Vendor return updated successfully",
  "data": { "...full VendorReturnResponse object..." }
}
```

**Frontend Impact:**
- Edit form for vendor returns in pending/shipped status

---

## PATCH /vendor-returns/:id/status
**Purpose:** Advance the status of a vendor return through its lifecycle
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric vendor return ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "status": "pending | shipped | received_by_vendor | completed"
}
```

**Expected Response 200:**
```json
{
  "message": "Vendor return status updated",
  "data": { "...full VendorReturnResponse with updated status..." }
}
```

**Frontend Impact:**
- Status progression buttons on return detail page
- `completedDate` set by backend when status moves to `"completed"`
- Inventory may be adjusted when status reaches `"received_by_vendor"` (backend-dependent)

---

## DELETE /vendor-returns/:id
**Purpose:** Delete a vendor return
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric vendor return ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Vendor return deleted successfully"
}
```

**Frontend Impact:**
- Return removed from list
- Should only be allowed for pending returns (client-side guard recommended)
