# Module Overview

**Module Name:** Inventory
**Module Path:** `inventory_management/backend/inventory`

This module provides a read-only, paginated view of current stock levels across all products and their variants, broken down by warehouse location. It is a computed view — there is no dedicated inventory table.

---

# Business Context

The Inventory module gives operators a consolidated, real-time snapshot of how much stock is available across all products and locations. It normalizes simple products and product variants into a flat row-per-SKU format, each row carrying a breakdown of quantities by warehouse. This view reflects the latest state of `variant_inventory`, which is updated by stock transfer and purchase operations.

---

# Functional Scope

## Included

- Read-only inventory view across all products and variants
- Per-location stock breakdown for each row
- Unified flat list: one row per simple product, one row per variant for variant products
- Pagination, search by product name or SKU, and optional filtering by location
- Scoped by `company_id` extracted from JWT

## Excluded

- Stock adjustment or write operations (handled by Stock Transfers and Purchase modules)
- Historical stock movement records
- Low-stock alerts or threshold configuration
- Inventory valuation or cost tracking

---

# Architecture Notes

- There is no dedicated `inventory` database table. The view is computed by joining:
  - `products` — for simple products
  - `product_variants` — for variant products
  - `variant_inventory` — per-location quantities per variant
  - `locations` — for location names
- Simple products contribute one row each. Variant products contribute one row per variant.
- The `stock` field on each row represents the total quantity across all locations for that product or variant.
- The `inventory` array on each row provides the per-location breakdown.
- `variantName` is an empty string for simple product rows.

---

# Data Model

**No dedicated table.** Computed from the following tables:

| Source Table        | Role                                                             |
|---------------------|------------------------------------------------------------------|
| `products`          | Product name, SKU, barcode, and type (simple vs variant)        |
| `product_variants`  | Variant SKU, barcode, and attributes for variant products        |
| `variant_inventory` | Per-location quantity records for each variant or simple product |
| `locations`         | Location names for the per-location breakdown                    |

**Inventory Row Shape:**

| Field          | Type     | Notes                                                              |
|----------------|----------|--------------------------------------------------------------------|
| `type`         | string   | `"product"` for simple products, `"variant"` for product variants  |
| `id`           | uint     | ID of the product (simple) or variant (variant)                   |
| `productId`    | uint     | ID of the parent product                                           |
| `productName`  | string   | Name of the product                                                |
| `variantName`  | string   | Variant label (e.g., `"Small / Red"`); empty string for simple products |
| `sku`          | string   | SKU of the product or variant                                      |
| `barcode`      | string   | Barcode; may be empty                                              |
| `stock`        | int      | Total quantity across all locations                                |
| `inventory`    | array    | Per-location breakdown; see structure below                        |

**Per-Location Inventory Entry:**

| Field          | Type   | Notes                          |
|----------------|--------|--------------------------------|
| `locationId`   | uint   | ID of the location             |
| `locationName` | string | Display name of the location   |
| `quantity`     | int    | Stock quantity at this location |

---

# API Endpoints

| Method | Path           | Auth       | Description                                           |
|--------|----------------|------------|-------------------------------------------------------|
| GET    | `/inventory/`  | Bearer JWT | Paginated inventory list with optional search and location filter |

---

# Request Payloads

### GET `/inventory/` — Query Parameters

| Parameter     | Type   | Default | Notes                                                        |
|---------------|--------|---------|--------------------------------------------------------------|
| `page`        | int    | `1`     |                                                              |
| `limit`       | int    | `10`    | Maximum: `100`                                               |
| `search`      | string | —       | LIKE match on product name or SKU                            |
| `location_id` | uint   | —       | When provided, filters to rows that have stock at this location |

---

# Response Contracts

### GET `/inventory/` — `200 OK`

```json
{
  "message": "Inventory retrieved successfully",
  "data": [
    {
      "type": "product",
      "id": 1,
      "productId": 1,
      "productName": "T-Shirt",
      "variantName": "",
      "sku": "TSHIRT-001",
      "barcode": "123456789",
      "stock": 100,
      "inventory": [
        { "locationId": 1, "locationName": "Main Warehouse", "quantity": 100 }
      ]
    },
    {
      "type": "variant",
      "id": 5,
      "productId": 2,
      "productName": "Jeans",
      "variantName": "32 / Blue",
      "sku": "JEANS-32-BLU",
      "barcode": "",
      "stock": 50,
      "inventory": [
        { "locationId": 1, "locationName": "Main Warehouse", "quantity": 30 },
        { "locationId": 2, "locationName": "Branch Store", "quantity": 20 }
      ]
    }
  ],
  "total": 25,
  "page": 1,
  "limit": 10
}
```

---

# Validation Rules

| Rule                            | HTTP Response |
|---------------------------------|---------------|
| Valid Bearer JWT required       | `401`         |
| `company_id` must be in context | `401`         |
| `limit` capped at `100`         | Enforced server-side |

---

# Business Logic Details

- `company_id` is always injected from JWT middleware. A missing `company_id` in context returns `401`.
- Each simple product produces exactly one inventory row. Each variant of a variant product produces one row per variant.
- The `stock` value is the sum of quantities across all `variant_inventory` records for that product/variant.
- The `inventory` array lists each location where the product/variant has a `variant_inventory` record, along with the quantity at that location.
- When `location_id` is provided as a filter, only rows that have stock entries at that location are returned.
- This endpoint is read-only. Inventory quantities are modified via Stock Transfers and other write modules.

---

# Dependencies

| Module / Table      | Direction  | Notes                                                           |
|---------------------|------------|-----------------------------------------------------------------|
| `products`          | Read       | Product name, SKU, barcode, type                               |
| `product_variants`  | Read       | Variant SKU, barcode, attribute labels                         |
| `variant_inventory` | Read       | Per-location quantities; updated by stock transfers and purchases |
| `locations`         | Read       | Location names for the per-location breakdown                  |
| `stock_transfers`   | Upstream   | Transfer operations update `variant_inventory`, reflected here |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is extracted from the JWT and used to scope all inventory queries; users cannot access inventory data belonging to other companies.
- No role-based access control is documented at the inventory module level (see Open Questions).
- This module is read-only; no write operations are exposed.

---

# Error Handling

| HTTP Status | Scenario                                         |
|-------------|--------------------------------------------------|
| `401`       | Missing or invalid JWT token                     |
| `401`       | `company_id` not found in request context        |
| `500`       | Unexpected server or database error              |

**Error response examples:**

```json
{ "error": "Company ID not found in context" }
{ "error": "Failed to retrieve inventory" }
```

---

# Testing Notes

- Test that simple products produce a single row with `type: "product"` and `variantName: ""`.
- Test that variant products produce one row per variant with `type: "variant"` and the correct `variantName`.
- Test that the `stock` field equals the sum of all `inventory[].quantity` values.
- Test search by product name (partial LIKE match).
- Test search by SKU (exact and partial LIKE match).
- Test `location_id` filter: rows without stock at the specified location must not appear.
- Test pagination (`page`, `limit`); confirm `limit` is capped at `100`.
- Test that a request with a missing or invalid JWT returns `401`.
- Test that inventory data from another company is not accessible.
- Confirm that a stock transfer updating `variant_inventory` is reflected immediately in subsequent `/inventory/` responses.

---

# Open Questions / Missing Details

- **`location_id` filter behavior:** It is unclear whether `location_id` filters rows to those that have any stock at the location, or whether it also filters the `inventory` array to show only that location's entry.
- **Zero-stock rows:** It is unclear whether products or variants with `stock = 0` are included in the response or excluded.
- **`stock` calculation source:** The documentation describes `stock` as a computed sum from `variant_inventory`. It is unclear whether this matches a `stock` field stored on the `products` or `product_variants` tables, or if it is always computed at query time.
- **Role-based access control:** No documentation on whether specific SaaS roles have differing permissions on the inventory endpoint.
- **Simple product inventory structure:** Simple products do not have variant records by definition. It is unclear whether their quantities are stored in `variant_inventory` with a placeholder variant or tracked directly on the `products` table.

---

# Change Log / Notes

- Initial documentation authored from source specification.
- Module is read-only; all stock write operations are managed by upstream modules (Stock Transfers, Purchases).
