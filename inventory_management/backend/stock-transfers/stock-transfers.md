# Module Overview

**Module Name:** Stock Transfers
**File Path:** `inventory_management/backend/stock-transfers/stock-transfers.md`
**Last Updated:** 2026-03-31

The Stock Transfers module manages the movement of inventory between warehouse locations. It supports transferring both simple product stock and variant-specific stock. Transfers are executed immediately — a transfer record is created with `status = Completed` on successful creation. Completed transfers can be cancelled, which reverses the stock movement. A helper endpoint is also provided to list products and their variant inventory at a specific location.

---

# Business Context

In a multi-location inventory system, stock must be physically moved between warehouses or stores. The Stock Transfers module provides the mechanism to record and execute these movements, ensuring that inventory counts remain accurate across all locations. Each transfer atomically adjusts the inventory at both the source and destination. If a transfer needs to be undone, cancellation reverses the stock changes exactly.

---

# Functional Scope

## Included

- Creating stock transfers between two different warehouse locations.
- Support for simple (non-variant) products and variant products.
- Immediate execution of transfers (no draft or pending approval flow — records are created as `Completed`).
- Cancellation of completed transfers with full stock reversal.
- Listing transfers with filtering by status, product, and location.
- Helper endpoint to query which products (and which variants) have stock at a specific location.
- Pagination and search on list endpoints.

## Excluded

- Multi-step approval workflows before transfer execution.
- Partial transfers (quantity must be specified in full).
- Transfers between the same source and destination location.
- Transfers for products with insufficient source stock (rejected with `400`).
- Direct manipulation of inventory records outside of transfer logic.

---

# Architecture Notes

- All endpoints are scoped to `company_id` extracted from the Bearer JWT.
- The `stock_transfers` table uses soft delete via `deleted_at`. Records are logically deleted but retained for audit trail.
- Transfers are ordered by `created_at DESC` by default.
- The `status` field in the DB schema defaults to `Pending`, but the application sets it to `Completed` immediately on a successful create. The `Pending` default exists at the DB level only as a safety fallback.
- Variant stock is tracked in a separate `variant_inventory` table (per variant per location). The `product_variants.stock` field is kept in sync by summing all `variant_inventory.quantity` values for that variant after each change.
- For simple products, stock is tracked directly on `products.stock` and `products.location_id`. When a simple product's full stock is transferred out, the product's `location_id` is reassigned to the destination.
- Cancellation reverses the exact quantity — no validation of current destination stock is documented (see Open Questions).
- The `GET /transfers/products-by-location/:location_id` endpoint filters variants to only those with existing inventory rows at the given location.

---

# Data Model

## Table: `stock_transfers`

| Column           | Type                                       | Constraints                             | JSON Key       |
|------------------|--------------------------------------------|-----------------------------------------|----------------|
| id               | uint                                       | PK, Auto Increment                      | id             |
| company_id       | uint                                       | Not null                                | companyId      |
| product_id       | uint                                       | FK → products.id, Not null              | productId      |
| variant_id       | *uint (nullable)                           | FK → product_variants.id                | variantId      |
| from_location_id | uint                                       | FK → locations.id, Not null             | fromLocationId |
| to_location_id   | uint                                       | FK → locations.id, Not null             | toLocationId   |
| quantity         | int                                        | Not null                                | quantity       |
| status           | enum('Pending','Completed','Cancelled')    | Default: `'Pending'`                    | status         |
| notes            | text                                       |                                         | notes          |
| created_at       | timestamp                                  | Auto-managed; default sort order DESC   | createdAt      |
| updated_at       | timestamp                                  | Auto-managed                            | updatedAt      |
| deleted_at       | timestamp (nullable)                       | Soft delete                             | —              |

## Relationships

| From          | Relationship | To              | Notes                             |
|---------------|--------------|-----------------|-----------------------------------|
| StockTransfer | BelongsTo    | Product         | Via `product_id`                  |
| StockTransfer | BelongsTo    | ProductVariant  | Via `variant_id`; optional        |
| StockTransfer | BelongsTo    | FromLocation    | Via `from_location_id`            |
| StockTransfer | BelongsTo    | ToLocation      | Via `to_location_id`              |

---

# API Endpoints

| Method | URL                                            | Auth       | Purpose                                                  |
|--------|------------------------------------------------|------------|----------------------------------------------------------|
| GET    | `/transfers/products-by-location/:location_id` | Bearer JWT | List products with stock at a specific location          |
| GET    | `/transfers/`                                  | Bearer JWT | List all transfers with pagination and filters           |
| POST   | `/transfers/`                                  | Bearer JWT | Execute a stock transfer between two locations           |
| PUT    | `/transfers/:id/cancel`                        | Bearer JWT | Cancel a completed transfer and reverse stock movement   |

---

# Request Payloads

## `GET /transfers/products-by-location/:location_id` — Path & Query Parameters

**Path Parameter:**

| Parameter   | Type | Required | Description                  |
|-------------|------|----------|------------------------------|
| location_id | uint | Yes      | ID of the warehouse location |

**Query Parameters:**

| Parameter | Type   | Required | Default | Description                                 |
|-----------|--------|----------|---------|---------------------------------------------|
| page      | int    | No       | 1       | Page number                                 |
| limit     | int    | No       | 50      | Records per page; max 100                   |
| search    | string | No       | —       | LIKE match on product name or SKU           |

---

## `GET /transfers/` — Query Parameters

| Parameter        | Type   | Required | Default | Description                                                                        |
|------------------|--------|----------|---------|------------------------------------------------------------------------------------|
| page             | int    | No       | 1       | Page number                                                                        |
| limit            | int    | No       | 10      | Records per page; max 100                                                          |
| status           | string | No       | —       | `Pending`, `Completed`, or `Cancelled`. Use `"all"` to skip filter.                |
| product_id       | uint   | No       | —       | Filter by product ID                                                               |
| from_location_id | uint   | No       | —       | Filter by source location ID                                                       |
| to_location_id   | uint   | No       | —       | Filter by destination location ID                                                  |

---

## `POST /transfers/` — Request Body

**Content-Type:** `application/json`

| Field          | Type   | Required | Description                                                                   |
|----------------|--------|----------|-------------------------------------------------------------------------------|
| productId      | uint   | Yes      | ID of the product being transferred                                           |
| variantId      | uint   | No       | ID of the product variant. Omit entirely for simple (non-variant) products.   |
| fromLocationId | uint   | Yes      | Source warehouse location ID                                                  |
| toLocationId   | uint   | Yes      | Destination warehouse location ID; must differ from `fromLocationId`          |
| quantity       | int    | Yes      | Number of units to transfer; must be at least 1                               |
| notes          | string | No       | Internal notes about the transfer                                             |

**Example (variant product):**
```json
{
  "productId": 2,
  "variantId": 5,
  "fromLocationId": 1,
  "toLocationId": 2,
  "quantity": 10,
  "notes": "Moving excess stock to branch"
}
```

**Example (simple product):**
```json
{
  "productId": 7,
  "fromLocationId": 1,
  "toLocationId": 3,
  "quantity": 20
}
```

---

# Response Contracts

## `GET /transfers/products-by-location/:location_id` — 200 OK

```json
{
  "message": "Products at location retrieved successfully",
  "location_id": 1,
  "data": [
    {
      "id": 2,
      "name": "T-Shirt",
      "sku": "TS-001",
      "variants": [
        { "id": 5, "name": "Small / Red", "stock": 15 }
      ]
    }
  ],
  "total": 10,
  "page": 1,
  "limit": 50
}
```

Only variants with an existing `variant_inventory` row for this location are included. Variants with zero or no inventory at this location are filtered out.

## `GET /transfers/` — 200 OK

```json
{
  "message": "Transfers retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "productId": 2,
      "product": { "id": 2, "name": "T-Shirt" },
      "variantId": 5,
      "variant": { "id": 5, "name": "Small / Red" },
      "fromLocationId": 1,
      "fromLocation": { "id": 1, "name": "Main Warehouse" },
      "toLocationId": 2,
      "toLocation": { "id": 2, "name": "Branch Store" },
      "quantity": 10,
      "status": "Completed",
      "notes": "Moving excess stock",
      "createdAt": "2026-03-01T09:00:00Z",
      "updatedAt": "2026-03-01T09:00:00Z"
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 10
}
```

## `POST /transfers/` — 201 Created

```json
{
  "message": "Transfer completed successfully",
  "data": { /* transfer object with all relations preloaded; status: "Completed" */ }
}
```

## `PUT /transfers/:id/cancel` — 200 OK

```json
{
  "message": "Transfer cancelled successfully",
  "data": { /* transfer object with status: "Cancelled" */ }
}
```

---

# Validation Rules

| Field          | Rule                                                                          |
|----------------|-------------------------------------------------------------------------------|
| productId      | Required; must reference an existing product within the company               |
| fromLocationId | Required                                                                      |
| toLocationId   | Required; must differ from `fromLocationId`                                   |
| quantity       | Required; must be an integer >= 1                                             |
| variantId      | Optional; if provided, variant-path logic is used; if omitted, simple-product logic is used |
| location_id    | Must be a valid parseable uint; returns `400` if invalid                      |
| status (cancel)| Transfer must currently have status `Completed`; returns `400` otherwise      |

---

# Business Logic Details

## CreateTransfer — Variant Product Path

Triggered when `variantId` is provided in the request.

1. Validate that `fromLocationId` and `toLocationId` are different. Return `400` if equal.
2. Look up the `variant_inventory` row for `(variant_id, from_location_id)`.
3. If no row is found: fall back to `product_variants.stock` if the parent product's `location_id` matches `from_location_id` — seed a new `variant_inventory` row using that stock value.
4. Check that available quantity is sufficient. Return `400` with `"Insufficient stock in source warehouse"` if not.
5. Deduct the transferred quantity from the source `variant_inventory` row.
6. Upsert the destination `variant_inventory` row: create if it does not exist; otherwise add the transferred quantity to the existing row.
7. Sync `product_variants.stock` = SUM of all `variant_inventory.quantity` for that variant across all locations.
8. Create a `stock_transfers` record with `status = "Completed"`.

## CreateTransfer — Simple Product Path

Triggered when `variantId` is not provided in the request.

1. Verify that the product's `location_id` matches `fromLocationId`. Return `400` or `404` if the product is not found.
2. Check that `products.stock` is sufficient. Return `400` if not.
3. Deduct the transferred quantity from `products.stock`.
4. If `products.stock` reaches 0 after deduction: update `products.location_id` to `toLocationId` and set `products.stock` to the transferred quantity.
5. Create a `stock_transfers` record with `status = "Completed"`.

## CancelTransfer

1. Look up the transfer by ID within the company scope. Return `404` if not found.
2. Validate the transfer's current status is `Completed`. Return `400` with `"Only completed transfers can be cancelled"` if not.
3. Reverse the stock movement: add quantity back to the source, deduct from the destination.
4. Update the transfer's `status` to `"Cancelled"`.

---

# Dependencies

| Module      | Direction | Notes                                                                                    |
|-------------|-----------|------------------------------------------------------------------------------------------|
| `products`  | Outbound  | `products.stock` and `products.location_id` are read and updated for simple product transfers |
| `locations` | Outbound  | `from_location_id` and `to_location_id` must reference valid `locations` records         |
| `inventory` | Outbound  | `variant_inventory` rows are read, created, and updated for variant product transfers; `product_variants.stock` is synced after each change |
| `companies` | Outbound  | All records scoped to `company_id` from JWT                                              |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` is extracted from the JWT and enforced on all queries.
- A `401` error is returned when `company_id` cannot be found in the JWT context.
- No additional role-level access control (RBAC) beyond JWT authentication is documented for this module.

---

# Error Handling

| HTTP Status | Trigger                                                                          |
|-------------|----------------------------------------------------------------------------------|
| 400         | Invalid `location_id` path parameter (non-integer)                               |
| 400         | Invalid request body (malformed JSON)                                            |
| 400         | `fromLocationId` equals `toLocationId`                                           |
| 400         | Insufficient stock at the source location or warehouse                           |
| 400         | Attempting to cancel a transfer that is not in `Completed` status                |
| 401         | `company_id` not found in JWT context                                            |
| 404         | Product not found for given `productId`                                          |
| 404         | Transfer not found for given `id`                                                |
| 500         | Unexpected server or database error                                              |

**Error response shapes:**
```json
{ "error": "Source and destination warehouse must be different" }
{ "error": "Insufficient stock in source warehouse" }
{ "error": "Only completed transfers can be cancelled" }
{ "error": "Product not found" }
{ "error": "Transfer not found" }
{ "error": "Failed to create transfer", "details": "..." }
{ "error": "Failed to cancel transfer", "details": "..." }
```

---

# Testing Notes

- **Variant transfer — happy path:** Create a variant transfer and verify that the source `variant_inventory` row is decremented, the destination row is created or incremented, and `product_variants.stock` is updated to the sum of all location inventories.
- **Variant transfer — fallback seed:** Test the case where no `variant_inventory` row exists for the source location but `product_variants.stock` matches the product's `location_id`. Confirm a new inventory row is seeded and the transfer proceeds.
- **Simple product transfer:** Verify that `products.stock` is decremented and the transfer record is created. Test the edge case where remaining stock reaches 0 — confirm `products.location_id` is reassigned and `products.stock` is set to the transferred quantity.
- **Insufficient stock:** Attempt a transfer with a quantity greater than available stock at the source. Confirm `400` is returned and no inventory is modified.
- **Same location rejection:** Submit a transfer with identical `fromLocationId` and `toLocationId`. Confirm `400` is returned immediately.
- **Cancellation — happy path:** Cancel a `Completed` transfer and verify stock is reversed at both source and destination.
- **Cancellation — invalid status:** Attempt to cancel a `Cancelled` or `Pending` (if one exists) transfer. Confirm `400` is returned.
- **Products-by-location filtering:** Confirm only variants with a `variant_inventory` row at the given location appear in the response. Variants without inventory at that location must be excluded.
- **Pagination and filtering:** Test `page`, `limit`, `status`, `product_id`, `from_location_id`, and `to_location_id` filters on `GET /transfers/`.
- **Company scoping:** Confirm transfers from other companies are not accessible.

---

# Open Questions / Missing Details

- **Simple product — partial transfer:** When a simple product is transferred partially (remaining stock > 0), `products.location_id` is not updated. This implies a simple product can only be at one location at a time. Confirm whether partial simple product transfers are intentionally restricted or if there is a gap in the logic.
- **Cancellation stock validation:** When cancelling a transfer, the reversal adds stock back to the source and deducts from the destination. It is unclear whether the destination is validated to have sufficient stock before the reversal deduction (to prevent negative inventory).
- **`Pending` status reachability:** The DB schema defines a `Pending` status but the application always creates transfers as `Completed`. Confirm whether any code path creates a `Pending` transfer, or whether this enum value is vestigial.
- **Variant fallback seed logic detail:** In the variant path, if the fallback `product_variants.stock` is used to seed a `variant_inventory` row, it is unclear whether the original `product_variants.stock` value is then zeroed out or left as-is.
- **`location_id` on products table:** The simple product path checks `products.location_id` against `fromLocationId`. The structure and semantics of `products.location_id` are not fully documented — confirm whether a simple product can only ever reside at one location at a time.
- **`product_variants.stock` sync on cancel:** When cancelling a variant transfer, it is unclear whether `product_variants.stock` is re-synced (SUM across all `variant_inventory` rows) in the same way as on create.
- **RBAC beyond JWT:** It is unknown whether specific role permissions restrict access to stock transfer endpoints beyond requiring a valid JWT.

---

# Change Log / Notes

| Date       | Author | Note                                                        |
|------------|--------|-------------------------------------------------------------|
| 2026-03-31 | System | Initial standardized documentation created from source spec |
