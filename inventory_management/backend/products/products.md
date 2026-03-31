# Module Overview

**Module Name:** Products
**Internal Identifier:** products
**Purpose:** Manages the product catalog. Products can be simple (single stock pool) or variant-based (distinct stock per variant per warehouse location). Supports file uploads for a main image and a gallery. Stock is managed through `products.stock` for simple products or `variant_inventory` for variant products, with automatic synchronization.

---

# Business Context

The product catalog is the central reference for all inventory, sales, and return operations. Each product belongs to a company and can be categorized, assigned to a vendor, and linked to a warehouse location. Variant products allow fine-grained stock management per size, color, or other attribute combinations, tracked per location in `variant_inventory`. Cost price is recorded per product and per variant to support profitability calculations in the Orders module.

---

# Functional Scope

## Included

- Full CRUD for products (with soft delete)
- Simple products (no variants) and variant products
- Per-variant inventory tracking by location (`variant_inventory`)
- Gallery image support (`product_images`) with position ordering
- Two-phase file commit (temp → final path) with rollback on DB failure
- Product attributes via many-to-many join table
- Stock auto-sync: `product_variants.stock` and `products.stock` kept in sync on every stock change
- Published/unpublished status toggle
- Filtering by category, vendor, and location on the list endpoint
- All operations scoped to authenticated company via JWT

## Excluded

- Direct stock adjustment API (stock changes occur via Orders, Customer Returns, Vendor Returns, and stock transfer modules)
- Bulk product import or CSV upload
- Product reviews or ratings

---

# Architecture Notes

- All queries are scoped by `company_id` extracted from the Bearer JWT.
- Product creation and update use `multipart/form-data` — there are no separate `-with-image` routes; image and data are always submitted together.
- Image handling uses a two-phase commit: files are saved to a temporary path first, then moved to their final path only after the database transaction succeeds. Temp files are rolled back on DB failure.
- Main image: single file in the `image` form field, stored in `./uploads/products/`, path saved in `products.image`.
- Gallery images: multiple files in the `images[]` form field, paths stored in `product_images`.
- New gallery images uploaded via PUT replace existing gallery images.
- `product_variants.stock` is always the authoritative sum of `variant_inventory.quantity` for that variant. `products.stock` is the sum of all variant stocks. Both are synced automatically on every stock change (orders, returns, transfers).
- For simple products, `products.stock` is managed directly.
- Unique constraints: `(company_id, sku)` on `products`; `(company_id, barcode)` on `products`; `(product_id, sku)` on `product_variants`.
- Product deletion is a soft delete (sets `deleted_at`). Associated image files are cleaned up from disk.

---

# Data Model

## Table: `products`

| Column           | Type         | Constraints / Notes                                           |
|------------------|--------------|---------------------------------------------------------------|
| `id`             | uint         | Primary key, auto-increment                                   |
| `company_id`     | uint         | FK → `companies.id`; indexed                                  |
| `name`           | varchar(255) | Not null                                                      |
| `description`    | text         |                                                               |
| `category_id`    | *uint        | FK → `categories.id`; nullable                                |
| `vendor_id`      | *uint        | FK → `users.id`; nullable                                     |
| `location_id`    | *uint        | FK → `locations.id`; nullable                                 |
| `price`          | float64      | Default: `0`                                                  |
| `sale_price`     | float64      | Default: `0`                                                  |
| `cost_price`     | float64      | Default: `0`; purchase/acquisition cost from vendor           |
| `stock`          | int          | Default: `0`; synced from variant stocks if variants exist    |
| `sku`            | varchar(100) | Unique per company; index: `idx_product_company_sku`          |
| `barcode`        | varchar(100) | Unique per company; index: `idx_product_company_barcode`      |
| `published`      | bool         | Default: `false`                                              |
| `receipt_number` | string       |                                                               |
| `image`          | string       | Path to main image file                                       |
| `uploaded_by`    | *uint        | FK → `saas_users.id`; nullable; who uploaded the main image   |
| `created_at`     | timestamp    | Auto-managed                                                  |
| `updated_at`     | timestamp    | Auto-managed                                                  |
| `deleted_at`     | timestamp    | Soft delete                                                   |

## Table: `product_variants`

| Column       | Type         | Constraints / Notes                                           |
|--------------|--------------|---------------------------------------------------------------|
| `id`         | uint         | Primary key                                                   |
| `product_id` | uint         | FK → `products.id`; unique with `sku`                         |
| `name`       | varchar(255) | e.g. `"Small / Red"`                                          |
| `attributes` | JSON         | e.g. `{"Size": "Small", "Color": "Red"}`                      |
| `price`      | float64      | Default: `0`                                                  |
| `sale_price` | float64      | Default: `0`                                                  |
| `cost_price` | float64      | Default: `0`; overrides `products.cost_price` if set          |
| `stock`      | int          | Synced from `SUM(variant_inventory.quantity)` for this variant|
| `sku`        | varchar(100) | Unique: index `idx_product_sku` on `(product_id, sku)`        |
| `barcode`    | varchar(100) | Indexed                                                       |
| `created_at` | timestamp    |                                                               |
| `updated_at` | timestamp    |                                                               |
| `deleted_at` | timestamp    | Soft delete                                                   |

## Table: `product_images`

| Column       | Type      | Constraints / Notes                    |
|--------------|-----------|----------------------------------------|
| `id`          | uint      | Primary key                                                  |
| `product_id`  | uint      | FK → `products.id`; CASCADE delete                           |
| `path`        | string    | File path                                                    |
| `position`    | int       | Default: `0`; ordering index                                 |
| `is_primary`  | bool      | Default: `false`                                             |
| `uploaded_by` | *uint     | FK → `saas_users.id`; nullable; who uploaded this image      |
| `created_at`  | timestamp |                                                              |
| `deleted_at`  | timestamp | Soft delete                                                  |

## Table: `variant_inventory`

| Column        | Type      | Constraints / Notes                          |
|---------------|-----------|----------------------------------------------|
| `id`          | uint      | Primary key                                  |
| `variant_id`  | uint      | FK → `product_variants.id`; CASCADE delete   |
| `location_id` | uint      | FK → `locations.id`                          |
| `quantity`    | int       | Default: `0`                                 |
| `created_at`  | timestamp |                                              |
| `updated_at`  | timestamp |                                              |
| `deleted_at`  | timestamp | Soft delete                                  |

## Join Table: `product_attributes`

| Column         | Notes                  |
|----------------|------------------------|
| `product_id`   | FK → `products.id`     |
| `attribute_id` | FK → `attributes.id`   |

## Relationships

| Relationship                                | Type          | Notes                                          |
|---------------------------------------------|---------------|------------------------------------------------|
| `Product` → `Category`                      | Belongs To    | Optional; nullable `category_id`               |
| `Product` → `User/Vendor`                   | Belongs To    | Optional; nullable `vendor_id` → `users.id`    |
| `Product` → `Location`                      | Belongs To    | Optional; nullable `location_id`               |
| `Product` ↔ `Attributes`                    | Many to Many  | Via `product_attributes` join table            |
| `Product` → `ProductVariants`               | Has Many      | CASCADE delete                                 |
| `Product` → `ProductImages`                 | Has Many      | CASCADE delete                                 |
| `ProductVariant` → `VariantInventory`       | Has Many      | Via `variant_id`; CASCADE delete               |
| `VariantInventory` → `Location`             | Belongs To    |                                                |

---

# API Endpoints

| Method | Path                       | Auth       | Description                                              |
|--------|----------------------------|------------|----------------------------------------------------------|
| GET    | `/products/`               | Bearer JWT | List products with pagination and filters                |
| GET    | `/products/:id`            | Bearer JWT | Get a single product with all relations                  |
| POST   | `/products/`               | Bearer JWT | Create a product with optional image/gallery (multipart) |
| PUT    | `/products/:id`            | Bearer JWT | Partial update with optional image/gallery (multipart)   |
| PATCH  | `/products/:id/status`     | Bearer JWT | Toggle published status                                  |
| DELETE | `/products/:id`            | Bearer JWT | Soft delete + clean up image files                       |

---

# Request Payloads

## POST `/products/` — Create Product

**Content-Type:** `multipart/form-data`

| Field            | Type    | Required | Notes                                           |
|------------------|---------|----------|-------------------------------------------------|
| `name`           | string  | Yes      |                                                 |
| `description`    | string  | No       |                                                 |
| `category_id`    | uint    | No       |                                                 |
| `vendor_id`      | uint    | No       |                                                 |
| `location_id`    | uint    | No       |                                                 |
| `price`          | float64 | No       |                                                 |
| `sale_price`     | float64 | No       |                                                 |
| `cost_price`     | float64 | No       | Purchase/acquisition cost                       |
| `stock`          | int     | No       | Initial stock for simple products               |
| `sku`            | string  | No       | Unique per company                              |
| `barcode`        | string  | No       | Unique per company                              |
| `published`      | bool    | No       |                                                 |
| `receipt_number` | string  | No       |                                                 |
| `attributes`     | string  | No       | JSON array of attribute IDs: `[1, 2, 3]`        |
| `variants`       | string  | No       | JSON array of variant objects (see schema below)|
| `image`          | file    | No       | Main product image                              |
| `images[]`       | file[]  | No       | Gallery images (multiple)                       |

**Variants JSON array format (value of `variants` field):**
```json
[
  {
    "name": "Small / Red",
    "attributes": {"Size": "Small", "Color": "Red"},
    "price": 29.99,
    "sale_price": 0,
    "cost_price": 15.00,
    "stock": 50,
    "sku": "TSHIRT-S-RED",
    "barcode": "",
    "inventory": [
      {"location_id": 1, "quantity": 50}
    ]
  }
]
```

## PUT `/products/:id` — Partial Update

**Content-Type:** `multipart/form-data`

Same fields as POST. All fields are optional. New images replace existing gallery images.

## PATCH `/products/:id/status` — Toggle Published Status

**Content-Type:** `application/json`

```json
{
  "published": true
}
```

---

# Response Contracts

## GET `/products/` — 200 OK

**Query parameters:**

| Param         | Type | Default | Description           |
|---------------|------|---------|-----------------------|
| `page`        | int  | 1       |                       |
| `limit`       | int  | 10      |                       |
| `category_id` | uint |         | Filter by category    |
| `vendor_id`   | uint |         | Filter by vendor      |
| `location_id` | uint |         | Filter by location    |

```json
{
  "message": "Products retrieved successfully",
  "page": 1,
  "limit": 10,
  "total": 50,
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "name": "T-Shirt",
      "description": "...",
      "category_id": 2,
      "vendor_id": null,
      "location_id": 1,
      "price": 29.99,
      "sale_price": 0,
      "cost_price": 15.00,
      "stock": 100,
      "sku": "TSHIRT-001",
      "barcode": "123456789",
      "published": true,
      "receipt_number": "",
      "image": "/uploads/products/img.jpg",
      "attributes": [],
      "variants": [],
      "images": [],
      "category": {},
      "vendor": {},
      "location": {},
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

## GET `/products/:id` — 200 OK

```json
{
  "message": "Product fetched successfully",
  "data": { /* full product object with all relations */ }
}
```

**400:** `{"error": "Invalid product ID"}`
**404:** `{"error": "Product not found"}`
**500:** `{"error": "Failed to fetch product"}`

## POST `/products/` — 201 Created

```json
{
  "message": "Product created successfully",
  "data": { /* full product object */ }
}
```

## PUT `/products/:id` — 200 OK

```json
{
  "message": "Product updated successfully",
  "data": { /* full product object */ }
}
```

## PATCH `/products/:id/status` — 200 OK

```json
{
  "message": "Product status updated successfully",
  "data": { /* full product object */ }
}
```

## DELETE `/products/:id` — 200 OK

```json
{
  "message": "Product deleted successfully"
}
```

---

# Validation Rules

| Field        | Rule                                                                     |
|--------------|--------------------------------------------------------------------------|
| `name`       | Required on create                                                       |
| `sku`        | Must be unique per company if provided; index: `(company_id, sku)`       |
| `barcode`    | Must be unique per company if provided; index: `(company_id, barcode)`   |
| `category_id`, `vendor_id`, `location_id` | Must reference existing records; HTTP 422 if invalid |
| `published`  | Required bool on PATCH status endpoint                                   |

Variant-level:
- `product_variants.sku` must be unique per product: index `(product_id, sku)`.

---

# Business Logic Details

### Product Creation Flow
1. Parse multipart form fields and optional image/gallery files.
2. Begin database transaction.
3. Create product record (obtains the product ID).
4. Create variant records with the correct `product_id`.
5. Seed `variant_inventory` records per variant per location.
6. Two-phase file commit: save images to temp paths; on DB success, move to final paths under `./uploads/products/`; on DB failure, delete temp files.

### Stock Management
- **Simple products:** `products.stock` is the direct stock count. It is decremented and incremented by other modules (Orders, Customer Returns, Vendor Returns).
- **Variant products:** `product_variants.stock = SUM(variant_inventory.quantity)` for that variant. `products.stock = SUM(all variant stocks)`. Both values are kept in sync automatically whenever any stock change occurs.
- If a product has variants, `products.stock` is derived and should not be set directly.

### Cost Price
- `products.cost_price` is the default cost for simple products or as a fallback.
- `product_variants.cost_price` overrides the product-level cost price for that specific variant.
- At order creation time, the applicable cost price is snapshotted into `order_items.unit_cost`.

### Image Lifecycle
- Main image path is stored in `products.image`.
- Gallery image paths are stored in `product_images.path`.
- Soft delete cleans up image files from disk.
- `product_images` and `product_variants` are CASCADE deleted when the product is hard-deleted (in DB).

---

# Dependencies

| Module       | Reason                                                                      |
|--------------|-----------------------------------------------------------------------------|
| `companies`  | `company_id` scoping from JWT                                               |
| `categories` | `category_id` FK; product categorization                                    |
| `locations`  | `location_id` FK on product; `location_id` FK in `variant_inventory`        |
| `users`      | `vendor_id` FK references `users.id` (vendor role)                          |
| `attributes` | Many-to-many via `product_attributes` join table                            |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` from the JWT scopes all queries. Users cannot access or modify products from another company.
- Unique constraints `(company_id, sku)` and `(company_id, barcode)` are enforced at the database level.

---

# Error Handling

| HTTP Status | Trigger                                                                               |
|-------------|---------------------------------------------------------------------------------------|
| 400         | Invalid product ID in path; `name` missing on create; invalid request body            |
| 404         | Product not found by ID                                                               |
| 422         | Invalid `category_id`, `vendor_id`, or `location_id` reference                       |
| 500         | Database error; file save failure; image cleanup failure                              |

---

# Testing Notes

- Verify `name` is required on POST (HTTP 400 if missing).
- Verify `(company_id, sku)` and `(company_id, barcode)` uniqueness: same SKU/barcode for different companies should succeed.
- Verify variant creation: variants are stored with correct `product_id`; `variant_inventory` rows are seeded correctly.
- Verify stock sync: after order creation, `product_variants.stock` and `products.stock` reflect the deducted quantities.
- Verify two-phase file commit: simulate DB failure and confirm temp files are cleaned up, no orphaned files remain.
- Verify gallery image replacement on PUT: old images are removed, new images are saved.
- Verify soft delete: product is not returned in list queries; image files are removed from disk.
- Verify `category_id`, `vendor_id`, `location_id` with non-existent IDs return HTTP 422.
- Verify `PATCH /products/:id/status` with invalid body returns HTTP 400.
- Test variant products: `products.stock` should equal the sum of all variant stocks.

---

# Open Questions / Missing Details

- Whether PUT replaces all variants or merges them is not specified; the source only says "partial update — only provided fields are changed" and "new images replace existing ones".
- The exact behavior when `stock` is provided on create for a product that also has variants is not documented.
- Whether `product_images.is_primary` is automatically set for the main image or is managed separately is not specified.
- Search/filtering on `name`, `sku`, or `barcode` is not mentioned for `GET /products/`.
- No `search` parameter is documented for the list endpoint.
- Whether `vendor_id` on `products` refers to `users.id` (with role_id=4 vendor) or a separate `vendors` table is not fully clear from the data model; the source states FK → `users.id`.

---

# Change Log / Notes

- Source documentation does not include a version history.
- This file was standardized from the original module notes on 2026-03-31.
