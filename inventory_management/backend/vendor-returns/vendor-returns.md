# Module Overview

**Module Name:** Vendor Returns
**Internal Identifier:** vendor-returns
**Purpose:** Manages product returns sent back to vendors or suppliers. Inventory is deducted immediately upon return creation (items are considered shipped back). Tracks a return status lifecycle and the type of credit received from the vendor.

---

# Business Context

When defective, surplus, or incorrectly shipped goods need to be sent back to a vendor, a vendor return is created. Unlike customer returns (which restock inventory on approval), vendor returns deduct inventory immediately at creation time — representing the physical removal of goods from the warehouse. The module tracks the return through a multi-step status flow (`pending` → `shipped` → `received_by_vendor` → `completed`) and records how the vendor credits the company (`refund`, `credit_note`, or `replacement`). A `completed_date` is set automatically when the return reaches `completed` status.

---

# Functional Scope

## Included

- Full CRUD for vendor returns (with soft delete)
- Multi-item returns (one return → many return items)
- Immediate inventory deduction on creation (transactional)
- Status lifecycle: `pending` → `shipped` → `received_by_vendor` → `completed`
- Dedicated status-only update endpoint (PATCH)
- `completed_date` auto-set when status transitions to `"completed"`
- Cost price snapshotting per item at return time
- Filtering and pagination on the list endpoint
- Aggregate statistics endpoint
- Returns listing scoped to a specific vendor
- All operations scoped to authenticated company via JWT

## Excluded

- Inventory restoration if a vendor return is deleted or cancelled
- Approval/rejection workflow (unlike customer returns, vendor returns have no approve/reject guards)
- Bulk return creation or import

---

# Architecture Notes

- All endpoints require Bearer JWT. `company_id` is extracted from the token and applied as a query scope.
- `return_number` defaults to `VRT-{unix_microseconds}` if not provided.
- `return_date` defaults to the current server timestamp if not provided.
- Inventory deduction runs inside a database transaction at creation. The return record and all item stock deductions are atomic.
- **Variant items:** deducted from `variant_inventory`. If no `variant_inventory` row exists for the variant, the deduction is skipped (no error).
- **Simple products:** deducted from `products.stock`. Returns HTTP 400 if stock is insufficient.
- `completed_date` is automatically set to the current timestamp when `status` becomes `"completed"` through either `PUT /vendor-returns/:id` or `PATCH /vendor-returns/:id/status`.
- List endpoint orders results by `return_date DESC`.
- `unit_cost` per item is snapshotted from `product_variants.cost_price` or `products.cost_price` at the time of return creation.

---

# Data Model

## Table: `vendor_returns`

| Column           | Type         | Constraints / Notes                                                            |
|------------------|--------------|--------------------------------------------------------------------------------|
| `id`             | uint         | Primary key, auto-increment                                                    |
| `company_id`     | uint         | FK → `companies.id`; indexed; JSON key: `companyId`                            |
| `return_number`  | varchar(100) | Unique; not null; JSON key: `returnNumber`                                     |
| `vendor_id`      | uint         | FK → `vendors.id`; not null; JSON key: `vendorId`                              |
| `vendor_name`    | varchar(255) | Not null; JSON key: `vendorName`                                               |
| `total_amount`   | float64      | Not null; default: `0`; JSON key: `totalAmount`                                |
| `status`         | string       | One of: `pending`, `shipped`, `received_by_vendor`, `completed`; default: `pending` |
| `return_date`    | timestamp    | Not null; ordered DESC; JSON key: `returnDate`                                 |
| `completed_date` | *timestamp   | Nullable; auto-set when status → `"completed"`; JSON key: `completedDate`      |
| `credit_type`    | string       | Not null; one of: `refund`, `credit_note`, `replacement`; JSON key: `creditType` |
| `notes`          | text         |                                                                                |
| `created_by`     | string       | JSON key: `createdBy`                                                          |
| `created_at`     | timestamp    | JSON key: `createdAt`                                                          |
| `updated_at`     | timestamp    | JSON key: `updatedAt`                                                          |
| `deleted_at`     | timestamp    | Soft delete                                                                    |

## Table: `vendor_return_items`

| Column         | Type    | Constraints / Notes                                                      |
|----------------|---------|--------------------------------------------------------------------------|
| `id`           | uint    | Primary key, auto-increment                                              |
| `return_id`    | uint    | FK → `vendor_returns.id`; not null                                       |
| `product_id`   | *uint   | FK → `products.id`; nullable                                             |
| `product_name` | string  | Not null; JSON key: `productName`                                        |
| `variant_id`   | *uint   | FK → `product_variants.id`; nullable                                     |
| `variant_name` | string  | JSON key: `variantName`                                                  |
| `quantity`     | int     | Not null; default: `1`                                                   |
| `unit_price`   | float64 | Not null; default: `0`; sale price at time of return; JSON key: `unitPrice` |
| `total_price`  | float64 | Not null; default: `0`; JSON key: `totalPrice`                           |
| `unit_cost`    | float64 | Default: `0`; `cost_price` snapshotted at return time; JSON key: `unitCost` |
| `reason`       | string  | Not null                                                                 |
| `created_at`   | timestamp |                                                                        |
| `updated_at`   | timestamp |                                                                        |

## Relationships

| Relationship                              | Type       | Notes                                 |
|-------------------------------------------|------------|---------------------------------------|
| `VendorReturn` → `Vendor`                 | Belongs To | Via `vendor_id`; required             |
| `VendorReturn` → `VendorReturnItems`      | Has Many   | Via `return_id`                       |
| `VendorReturnItem` → `Product`            | Belongs To | Optional; nullable `product_id`       |
| `VendorReturnItem` → `ProductVariant`     | Belongs To | Optional; nullable `variant_id`       |

---

# API Endpoints

| Method | Path                                   | Auth       | Description                                              |
|--------|----------------------------------------|------------|----------------------------------------------------------|
| GET    | `/vendor-returns/`                     | Bearer JWT | List returns with pagination, search, and filters        |
| GET    | `/vendor-returns/:id`                  | Bearer JWT | Get a single return with vendor and items                |
| POST   | `/vendor-returns/`                     | Bearer JWT | Create a return and immediately deduct inventory         |
| PUT    | `/vendor-returns/:id`                  | Bearer JWT | Update status and/or notes                               |
| PATCH  | `/vendor-returns/:id/status`           | Bearer JWT | Update status only                                       |
| DELETE | `/vendor-returns/:id`                  | Bearer JWT | Soft delete a return                                     |
| GET    | `/vendor-returns/stats`                | Bearer JWT | Get aggregate counts and total credit amount             |
| GET    | `/vendor-returns/vendor/:vendorId`     | Bearer JWT | List all returns for a specific vendor                   |

---

# Request Payloads

## POST `/vendor-returns/` — Create Vendor Return

**Content-Type:** `application/json`

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
      "unitCost": 10.00,
      "reason": "Defective batch"
    }
  ]
}
```

| Field          | Required | Notes                                                                  |
|----------------|----------|------------------------------------------------------------------------|
| `returnNumber` | No       | Auto-generated as `VRT-{unix_microseconds}` if empty                  |
| `vendorId`     | No       | FK → `vendors.id`                                                      |
| `vendorName`   | Yes      |                                                                        |
| `totalAmount`  | No       | Default: `0`                                                           |
| `status`       | No       | Default: `pending`; one of: `pending`, `shipped`, `received_by_vendor`, `completed` |
| `returnDate`   | No       | Defaults to current timestamp                                          |
| `creditType`   | Yes      | One of: `refund`, `credit_note`, `replacement`                        |
| `notes`        | No       |                                                                        |
| `items`        | No       | Array of return line items; inventory deducted for each               |
| `items[].productId`   | No  |                                                                    |
| `items[].productName` | Yes |                                                                    |
| `items[].variantId`   | No  |                                                                    |
| `items[].variantName` | No  |                                                                    |
| `items[].quantity`    | No  | Default: `1`                                                       |
| `items[].unitPrice`   | No  | Sale price at time of return                                       |
| `items[].totalPrice`  | No  |                                                                    |
| `items[].unitCost`    | No  | Can be provided explicitly or snapshotted from product/variant     |
| `items[].reason`      | Yes |                                                                    |

## PUT `/vendor-returns/:id` — Update Status and/or Notes

**Content-Type:** `application/json`

```json
{
  "status": "shipped",
  "notes": "Shipped via DHL"
}
```

`status` must be one of the valid status values if provided. Sets `completed_date` automatically when status → `"completed"`.

## PATCH `/vendor-returns/:id/status` — Update Status Only

**Content-Type:** `application/json`

```json
{
  "status": "completed"
}
```

`status` is required.

---

# Response Contracts

## GET `/vendor-returns/` — 200 OK

**Query parameters:**

| Param       | Type   | Default | Description                                                               |
|-------------|--------|---------|---------------------------------------------------------------------------|
| `page`      | int    | 1       |                                                                           |
| `per_page`  | int    | 10      |                                                                           |
| `search`    | string |         | LIKE match on `return_number`, `vendor_name`                              |
| `status`    | string |         | One of: `pending`, `shipped`, `received_by_vendor`, `completed`; `"all"` = no filter |
| `vendor_id` | uint   |         | Filter by vendor                                                          |

```json
{
  "message": "Vendor returns retrieved successfully",
  "data": [ /* array of return objects with vendor and items */ ],
  "pagination": {"page": 1, "per_page": 10, "total": 8}
}
```

## GET `/vendor-returns/:id` — 200 OK

```json
{
  "message": "Vendor return fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "returnNumber": "VRT-1700000000000",
    "vendorId": 2,
    "vendor": {},
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
        "unitCost": 10.00,
        "reason": "Defective batch"
      }
    ],
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**404:** `{"error": "Vendor return not found"}`

## POST `/vendor-returns/` — 201 Created

```json
{
  "message": "Vendor return created and inventory deducted successfully",
  "data": { /* return object with vendor and items */ }
}
```

## PUT `/vendor-returns/:id` — 200 OK

```json
{
  "message": "Vendor return updated successfully",
  "data": { /* return object with vendor and items */ }
}
```

## PATCH `/vendor-returns/:id/status` — 200 OK

```json
{
  "message": "Status updated successfully",
  "data": { /* return object with vendor and items */ }
}
```

## DELETE `/vendor-returns/:id` — 200 OK

```json
{
  "message": "Vendor return deleted successfully"
}
```

## GET `/vendor-returns/stats` — 200 OK

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

## GET `/vendor-returns/vendor/:vendorId` — 200 OK

```json
{
  "message": "Vendor returns retrieved successfully",
  "data": [ /* array of return objects for vendor, ordered by returnDate DESC */ ]
}
```

---

# Validation Rules

| Field          | Rule                                                                              |
|----------------|-----------------------------------------------------------------------------------|
| `vendorName`   | Required                                                                          |
| `creditType`   | Required; must be one of: `refund`, `credit_note`, `replacement`                 |
| `status`       | Must be one of: `pending`, `shipped`, `received_by_vendor`, `completed`           |
| `returnNumber` | Must be unique; HTTP 409 returned if duplicate                                    |
| `items[].reason` | Required per item                                                               |
| `items[].productName` | Required per item                                                          |
| Stock          | Insufficient stock for simple products returns HTTP 400 during creation           |

On PATCH: `status` is required.

---

# Business Logic Details

### Return Number Generation
If `returnNumber` is not provided (or is empty), the system auto-generates one as `VRT-{unix_microseconds}`. If provided, it must be unique; a duplicate returns HTTP 409.

### Return Date Defaulting
If `returnDate` is not provided, it defaults to the current server timestamp at creation time.

### Inventory Deduction on Create (Transactional)
All inventory deductions for a return run inside a single database transaction. The return record and all deductions are committed atomically.

- **Variant items:** Deduct the item quantity from the corresponding `variant_inventory` row for the variant. If no `variant_inventory` row exists for the variant, the deduction is **skipped** (no error; items are still created).
- **Simple products:** Deduct from `products.stock` directly. If stock is insufficient, returns HTTP 400 with an error message, and the entire transaction is rolled back.

### `completed_date` Auto-set
When a return's `status` is set to `"completed"` (via either `PUT /vendor-returns/:id` or `PATCH /vendor-returns/:id/status`), `completed_date` is automatically set to the current server timestamp. It is not set for any other status transition.

### Deletion
Soft delete only. **Inventory is not restored** when a vendor return is deleted.

---

# Dependencies

| Module      | Reason                                                                        |
|-------------|-------------------------------------------------------------------------------|
| `companies` | `company_id` scoping from JWT                                                 |
| `vendors`   | `vendor_id` FK; vendor data preloaded on fetch                                |
| `products`  | `products.stock` deducted for simple products; `variant_inventory` deducted for variant items |
| `product_variants` | `variant_inventory` rows targeted for deduction; cost price snapshotted |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` from the JWT scopes all queries. Users cannot access or modify vendor returns from another company.
- No additional role-based access control is documented at the endpoint level beyond JWT authentication.

---

# Error Handling

| HTTP Status | Trigger                                                                                           |
|-------------|---------------------------------------------------------------------------------------------------|
| 400         | Invalid request body; missing `vendorName`; invalid `status`; invalid `creditType`; insufficient stock for a simple product |
| 404         | Vendor return not found by ID                                                                     |
| 409         | Duplicate `return_number`                                                                        |
| 500         | Database error; failed to create vendor return                                                    |

**Error response examples:**
```json
{"error": "Vendor name is required"}
{"error": "Invalid status"}
{"error": "Invalid credit type"}
{"error": "insufficient inventory for variant ..."}
{"error": "insufficient stock for product ..."}
{"error": "Return number already exists"}
{"error": "Status is required"}
```

---

# Testing Notes

- Verify `return_number` auto-generation produces a unique `VRT-{unix_microseconds}` string.
- Verify HTTP 409 on duplicate `return_number`.
- Verify inventory deduction is transactional: if any item causes a stock error, the entire return creation rolls back.
- Verify variant item deduction: `variant_inventory.quantity` decreases by the item quantity.
- Verify variant item with no `variant_inventory` row: deduction is skipped, return is still created successfully.
- Verify simple product deduction: `products.stock` decreases; insufficient stock returns HTTP 400.
- Verify `completed_date` is set automatically when status → `"completed"` via both PUT and PATCH endpoints.
- Verify `completed_date` is **not** set for `shipped` or `received_by_vendor` status transitions.
- Verify soft delete does not restore inventory.
- Verify `status` filter on `GET /vendor-returns/`: `"all"` returns all statuses, specific value filters correctly.
- Verify `GET /vendor-returns/stats` totals are consistent with actual return records.
- Verify `GET /vendor-returns/vendor/:vendorId` returns only returns for that vendor.

---

# Open Questions / Missing Details

- The `stats` response includes `pending`, `shipped`, and `completed` but does not include `received_by_vendor`. Whether this is intentional or a gap in the source documentation is unclear.
- Whether `GET /vendor-returns/vendor/:vendorId` respects the company scope (i.e., only returns from the authenticated company for that vendor) is not explicitly stated; assumed to be scoped.
- The behavior when `unit_cost` is provided explicitly in the item payload vs. being snapshotted from the product/variant is not documented (which takes precedence).
- Whether `products.stock` is re-synced from `variant_inventory` after variant deduction (as it is in the Orders module) is not specified.
- Whether deleting a vendor return is allowed at any status, or only at certain statuses (e.g., only `pending`), is not documented.
- No validation is documented for `quantity > 0` or `unitPrice >= 0` on items.

---

# Change Log / Notes

- Source documentation does not include a version history.
- This file was standardized from the original module notes on 2026-03-31.
