# Module Overview

**Module Name:** Coupons
**Internal Identifier:** coupons
**Purpose:** Manages discount coupon campaigns for a company's storefront. Supports creating, reading, updating, soft-deleting, and validating coupons. Covers multiple discount types, usage controls, product/category applicability filters, auto-apply, and stackable coupon behavior. Includes a checkout validation endpoint and a usage statistics endpoint.

---

# Business Context

Companies create coupon campaigns to offer discounts to customers at checkout. Coupons are identified by a unique code per company and can be distributed freely. The system supports restricting coupons by date range, minimum order amount, total usage count, per-user usage count, and scope (specific products or categories). Auto-apply and stackable flags allow the storefront to resolve multiple applicable coupons automatically. A public lookup endpoint exists for storefront use without requiring authentication.

---

# Functional Scope

## Included

- Full CRUD for coupons (with soft delete)
- Optional image upload per coupon stored on disk
- Three discount types: `percentage`, `fixed`, `free_shipping`
- Usage tracking via `coupon_usages` table (linked to customers and orders)
- Per-coupon usage statistics
- Public coupon lookup by code for storefront (no auth required)
- Checkout validation endpoint: active status, date range, order minimum, usage limits, per-user limits, product/category applicability, calculated discount
- `auto_apply` and `stackable` flags for storefront coupon resolution
- `priority` field for ordering coupon resolution
- All management operations scoped to the authenticated company via JWT

## Excluded

- Coupon application to orders (handled by the Orders module)
- Bulk coupon generation or CSV import
- Customer-facing coupon management
- Coupon analytics beyond usage stats

---

# Architecture Notes

- All management endpoints require a Bearer JWT. `company_id` is extracted from the token and applied as a query scope on every operation.
- `GET /coupons/code/:code` is a public endpoint with no authentication, intended for storefront use. Whether it applies any company scoping is not specified in the source.
- Image handling is separated into dedicated `-with-image` endpoints (`multipart/form-data`). The standard JSON endpoints do not accept file uploads.
- On coupon deletion, the image file is removed from disk alongside the soft-delete database record.
- Coupon codes are unique per company, enforced by the database index `idx_coupon_company_code`.
- `max_discount` caps the computed discount amount for `percentage` type coupons (e.g., 20% off but no more than $15).
- `applicable_to_categories` and `applicable_to_products` store comma-separated IDs as text. An empty string means the coupon applies to all products/categories.
- Coupon validation logic lives in a service layer (`services.ValidateCoupon`). The handler delegates to it.

---

# Data Model

## Table: `coupons`

| Column                     | Type         | Constraints / Notes                                              |
|----------------------------|--------------|------------------------------------------------------------------|
| `id`                       | uint         | Primary key, auto-increment                                      |
| `company_id`               | uint         | FK → `companies.id`; indexed; JSON key: `companyId`              |
| `campaign_name`            | varchar(255) |                                                                  |
| `code`                     | varchar(100) | Unique per company; index: `idx_coupon_company_code`             |
| `discount`                 | float64      |                                                                  |
| `type`                     | varchar(50)  | One of: `percentage`, `fixed`, `free_shipping`                   |
| `start_date`               | timestamp    |                                                                  |
| `end_date`                 | timestamp    | Must be after `start_date`                                       |
| `status`                   | bool         | Default: `false` (inactive)                                      |
| `image`                    | string       | File path on disk                                                |
| `uploaded_by`              | *uint        | FK → `saas_users.id`; nullable; who uploaded the coupon image    |
| `usage_limit`              | *int         | Nullable = unlimited total uses                                  |
| `usage_limit_per_user`     | *int         | Nullable = unlimited per-user uses                               |
| `times_used`               | int          | Default: `0`; incremented on each valid use                      |
| `min_order_amount`         | float64      | Default: `0`                                                     |
| `max_discount`             | *float64     | Nullable; caps the computed discount for `percentage` type       |
| `applicable_to_categories` | text         | Comma-separated category IDs; empty = all categories             |
| `applicable_to_products`   | text         | Comma-separated product IDs; empty = all products                |
| `free_shipping`            | bool         | Default: `false`                                                 |
| `stackable`                | bool         | Default: `false`                                                 |
| `auto_apply`               | bool         | Default: `false`                                                 |
| `priority`                 | int          | Default: `0`; used for coupon resolution ordering                |
| `created_at`               | timestamp    | Auto-managed                                                     |
| `updated_at`               | timestamp    | Auto-managed                                                     |
| `deleted_at`               | timestamp    | Soft delete                                                      |

## Table: `coupon_usages`

| Column             | Type      | Constraints / Notes                                    |
|--------------------|-----------|--------------------------------------------------------|
| `id`               | uint      | Primary key, auto-increment                            |
| `coupon_id`        | uint      | FK → `coupons.id`                                      |
| `customer_id`      | *uint     | FK → `customers.id`; nullable                          |
| `sell_id`          | *uint     | FK → `sells.id`; nullable                              |
| `coupon_code`      | string    | Snapshot of code at time of use                        |
| `discount_applied` | float64   | Actual discount amount applied to the order            |
| `original_amount`  | float64   | Order amount before discount                           |
| `final_amount`     | float64   | Order amount after discount                            |
| `used_at`          | timestamp |                                                        |
| `created_at`       | timestamp |                                                        |
| `deleted_at`       | timestamp | Soft delete                                            |

## Relationships

| Relationship              | Type       | Notes                                 |
|---------------------------|------------|---------------------------------------|
| `Coupon` → `CouponUsages` | Has Many   |                                       |
| `CouponUsage` → `Customer`| Belongs To | Optional; nullable `customer_id`      |
| `CouponUsage` → `Sell`    | Belongs To | Optional; nullable `sell_id`          |

---

# API Endpoints

| Method | Path                       | Auth          | Description                                    |
|--------|----------------------------|---------------|------------------------------------------------|
| GET    | `/coupons/`                | Bearer JWT    | List all coupons for the authenticated company |
| GET    | `/coupons/:id`             | Bearer JWT    | Get a single coupon by ID                      |
| GET    | `/coupons/code/:code`      | None (public) | Look up an active coupon by code (storefront)  |
| POST   | `/coupons/`                | Bearer JWT    | Create a coupon (JSON)                         |
| POST   | `/coupons/with-image`      | Bearer JWT    | Create a coupon with image upload (multipart)  |
| PUT    | `/coupons/:id`             | Bearer JWT    | Partial update of a coupon (JSON)              |
| PUT    | `/coupons/:id/with-image`  | Bearer JWT    | Partial update with optional image (multipart) |
| DELETE | `/coupons/:id`             | Bearer JWT    | Soft delete + remove image file from disk      |
| POST   | `/coupons/validate`        | Bearer JWT    | Validate a coupon at checkout                  |
| GET    | `/coupons/:id/usage-stats` | Bearer JWT    | Get usage statistics for a coupon              |

---

# Request Payloads

## POST `/coupons/` — Create Coupon

**Content-Type:** `application/json`

```json
{
  "campaign_name": "Summer Sale",
  "code": "SUMMER20",
  "discount": 20,
  "type": "percentage",
  "start_date": "2024-06-01T00:00:00Z",
  "end_date": "2024-08-31T23:59:59Z",
  "status": true,
  "usage_limit": 100,
  "usage_limit_per_user": 1,
  "min_order_amount": 50,
  "max_discount": 30,
  "applicable_to_categories": "",
  "applicable_to_products": "",
  "free_shipping": false,
  "stackable": false,
  "auto_apply": false,
  "priority": 0
}
```

## POST `/coupons/with-image` — Create Coupon with Image

**Content-Type:** `multipart/form-data`

Same fields as `POST /coupons/`, plus:
- `image` — image file; saved to `uploads/coupons/`
- `start_date` and `end_date` must be RFC3339 formatted strings

## PUT `/coupons/:id` — Partial Update

**Content-Type:** `application/json`

Any subset of the create fields. All fields are optional (pointer types in `UpdateCouponRequest`).

## PUT `/coupons/:id/with-image` — Partial Update with Image

**Content-Type:** `multipart/form-data`

Any subset of: `campaign_name`, `code`, `type`, `status`, `start_date` (RFC3339), `end_date` (RFC3339), `image` (file). If a new image is provided, the old image file is deleted from disk. Returns a map of only the updated fields.

## POST `/coupons/validate` — Validate Coupon at Checkout

**Content-Type:** `application/json`

```json
{
  "code": "SUMMER20",
  "customerId": 5,
  "orderAmount": 120.00,
  "cartItems": [
    {
      "product_id": 2,
      "category_id": 1,
      "price": 60.00,
      "quantity": 2
    }
  ]
}
```

| Field                     | Type    | Required | Notes                                    |
|---------------------------|---------|----------|------------------------------------------|
| `code`                    | string  | Yes      |                                          |
| `customerId`              | uint    | No       | Used for per-user limit check            |
| `orderAmount`             | float64 | Yes      | Must be `> 0`                            |
| `cartItems`               | array   | No       | Used for product/category applicability  |
| `cartItems[].product_id`  | uint    | Yes      | Required when items are provided         |
| `cartItems[].category_id` | uint    | No       |                                          |
| `cartItems[].price`       | float64 | Yes      | Must be `> 0`                            |
| `cartItems[].quantity`    | int     | Yes      | Must be `> 0`                            |

---

# Response Contracts

## GET `/coupons/` — 200 OK

```json
{
  "message": "Coupons retrieved successfully",
  "data": [ /* array of coupon objects */ ]
}
```

## GET `/coupons/:id` — 200 OK

```json
{
  "message": "Coupon fetched successfully",
  "data": { /* coupon object */ }
}
```

**404:** `{"error": "Coupon not found"}`

## GET `/coupons/code/:code` — 200 OK

```json
{
  "message": "Coupon found",
  "data": { /* coupon object */ }
}
```

**404:** `{"error": "Coupon not found or inactive"}`

## POST `/coupons/` — 201 Created

```json
{
  "message": "Coupon created successfully",
  "data": { /* coupon object */ }
}
```

## POST `/coupons/validate` — 200 OK (valid coupon)

```json
{
  "valid": true,
  "discountAmount": 24.00
}
```

## POST `/coupons/validate` — 422 Unprocessable Entity (invalid coupon)

```json
{
  "valid": false,
  "error_code": "COUPON_EXPIRED",
  "message": "This coupon has expired"
}
```

## GET `/coupons/:id/usage-stats` — 200 OK

```json
{
  "message": "Usage statistics retrieved successfully",
  "data": { /* usage stats object from service layer */ }
}
```

**400:** `{"error": "Invalid coupon ID"}`
**404:** `{"error": "Coupon not found"}`

## DELETE `/coupons/:id` — 200 OK

```json
{
  "message": "Coupon deleted successfully"
}
```

---

# Validation Rules

Applied to `CreateCouponRequest`. Update requests apply the same rules for any fields that are provided.

| Field                      | Rule                                                       |
|----------------------------|------------------------------------------------------------|
| `campaign_name`            | Required; min length 3; max length 200                     |
| `code`                     | Required; min length 3; max length 50; alphanumeric only   |
| `discount`                 | Required; must be `> 0`                                    |
| `type`                     | Required; one of: `percentage`, `fixed`, `free_shipping`   |
| `start_date`               | Required                                                   |
| `end_date`                 | Required; must be after `start_date`                       |
| `status`                   | Optional bool                                              |
| `usage_limit`              | Optional `*int`                                            |
| `usage_limit_per_user`     | Optional `*int`                                            |
| `min_order_amount`         | Optional float64                                           |
| `max_discount`             | Optional `*float64`                                        |
| `applicable_to_categories` | Optional string                                            |
| `applicable_to_products`   | Optional string                                            |
| `free_shipping`            | Optional bool                                              |
| `stackable`                | Optional bool                                              |
| `auto_apply`               | Optional bool                                              |
| `priority`                 | Optional int                                               |

**Additional validations (service/handler layer):**
- `code` must be unique within the company (checked against `idx_coupon_company_code`)
- `end_date` must be after `start_date`; returns HTTP 400 if violated
- Multipart endpoints: `start_date` and `end_date` must parse as RFC3339; HTTP 400 on parse failure
- `PUT /coupons/:id/with-image`: at least one field must be provided; HTTP 400 if no fields present

---

# Business Logic Details

### Coupon Code Uniqueness
Codes are unique per company. The index `idx_coupon_company_code` enforces this at the database level.

### Image Lifecycle
- **Create with image:** File saved to `uploads/coupons/`.
- **Update with image:** Old image file deleted from disk; new file saved.
- **Delete coupon:** Associated image file removed from disk before or alongside the soft delete.

### Public Coupon Lookup (`GET /coupons/code/:code`)
- Looks up coupon by code via the service layer.
- Returns `404` if the coupon does not exist or `status = false` (inactive).
- No authentication required. Intended for storefront display.

### Coupon Validation (`POST /coupons/validate`)
Validation is performed in the following order:
1. Coupon exists and `status = true`
2. Current date/time is within `start_date` and `end_date`
3. `orderAmount >= min_order_amount`
4. `times_used < usage_limit` (if `usage_limit` is set)
5. Per-user usage count `< usage_limit_per_user` (if set and `customerId` is provided)
6. Cart items match `applicable_to_products` or `applicable_to_categories` (if restrictions are set)
7. Returns calculated `discountAmount` on success

### Discount Calculation
- **`percentage`:** `discountAmount = orderAmount × (discount / 100)`, capped at `max_discount` if `max_discount` is set.
- **`fixed`:** `discountAmount = discount` value directly.
- **`free_shipping`:** Applies to the order's shipping cost. Exact interaction with order totals is handled by the Orders module.

### `auto_apply` and `stackable` Flags
These are metadata flags consumed by the storefront. Whether they are enforced by the validation endpoint is not specified in the source.

---

# Dependencies

| Module       | Reason                                                              |
|--------------|---------------------------------------------------------------------|
| `companies`  | `company_id` scoping from JWT                                       |
| `customers`  | `coupon_usages.customer_id` references `customers.id`              |
| `orders`     | `coupon_usages.sell_id` references `sells.id`                      |
| `categories` | `applicable_to_categories` filter evaluated during validation       |
| `products`   | `applicable_to_products` filter evaluated during validation         |

---

# Security / Permissions

- All endpoints except `GET /coupons/code/:code` require a valid Bearer JWT.
- `company_id` is extracted from the JWT and scopes all queries. Users cannot access or modify coupons belonging to another company.
- `GET /coupons/code/:code` is unauthenticated. Only active coupons (`status = true`) are returned.
- The `password` field is not part of this module.

---

# Error Handling

| HTTP Status | Trigger                                                                              |
|-------------|--------------------------------------------------------------------------------------|
| 400         | `end_date` before `start_date`; invalid RFC3339 date on multipart endpoints; no update fields on `PUT /with-image` |
| 404         | Coupon not found by ID; coupon not found or inactive on public code lookup           |
| 422         | Request body validation failure; coupon validation failure at checkout               |
| 500         | Database error; image save failure; image delete failure                             |

Checkout validation failures return HTTP 422 with a structured body:
```json
{ "valid": false, "error_code": "...", "message": "..." }
```

---

# Testing Notes

- Verify `code` uniqueness is per company: the same code for two different companies should succeed.
- Verify `end_date` before `start_date` on both POST and PUT returns HTTP 400.
- Verify `GET /coupons/code/:code` with an inactive coupon (`status = false`) returns 404.
- Test each failure condition on `POST /coupons/validate`: expired, below minimum amount, total usage exceeded, per-user usage exceeded, not applicable to provided cart items.
- Verify `max_discount` cap is applied correctly for `percentage` type coupons.
- Verify image deletion from disk on coupon delete.
- Verify old image deletion and new image save on `PUT /coupons/:id/with-image`.
- Test multipart endpoints with invalid RFC3339 date formats return HTTP 400.
- Verify `times_used` increments after a successful order applies the coupon (via `coupon_usages` record creation in the Orders module).

---

# Open Questions / Missing Details

- Full list of `error_code` values returned by `POST /coupons/validate` is not documented.
- Exact response schema for `GET /coupons/:id/usage-stats` is not specified.
- Whether `GET /coupons/code/:code` applies any company scoping is not specified.
- Exact behavior when `free_shipping` type coupon interacts with order total calculation is not documented here (deferred to Orders module).
- Whether `auto_apply` and `stackable` flags are enforced by the validation endpoint or are purely frontend metadata is not specified.
- Pagination behavior for `GET /coupons/` is not documented (no `page` or `per_page` parameters listed in the source).
- Response body shape for `PUT /coupons/:id/with-image` ("returns partial update map") is not fully specified beyond being a map of changed fields.

---

# Change Log / Notes

- Source documentation does not include a version history.
- This file was standardized from the original module notes on 2026-03-31.
