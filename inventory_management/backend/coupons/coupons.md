# Module: Coupons

## Business Purpose
Manages discount coupon campaigns. Coupons support multiple discount types (percentage, fixed, free shipping), usage limits, order minimums, applicability rules (by product or category), and advanced features like auto-apply and stackable coupons. Includes a validation endpoint for checkout use and usage statistics. Supports both JSON and multipart/form-data (with image upload).

---

## Database Table: `coupons`

| Column                    | Type           | Notes                                                          |
|---------------------------|----------------|----------------------------------------------------------------|
| id                        | uint (PK, AI)  | json: id                                                       |
| company_id                | uint (FK)      | FK → companies.id; indexed; json: companyId                    |
| campaign_name             | varchar(255)   | json: campaign_name                                            |
| code                      | varchar(100)   | unique per company: idx_coupon_company_code; json: code        |
| discount                  | float64        | json: discount                                                 |
| type                      | varchar(50)    | 'percentage', 'fixed', 'free_shipping'; json: type             |
| start_date                | timestamp      | json: start_date                                               |
| end_date                  | timestamp      | json: end_date                                                 |
| status                    | bool           | default false; json: status                                    |
| image                     | string         | file path; json: image                                         |
| usage_limit               | *int           | nullable = unlimited; json: usage_limit                        |
| usage_limit_per_user      | *int           | nullable = unlimited; json: usage_limit_per_user               |
| times_used                | int            | default 0; json: times_used                                    |
| min_order_amount          | float64        | default 0; json: min_order_amount                              |
| max_discount              | *float64       | nullable; cap for percentage discounts; json: max_discount      |
| applicable_to_categories  | text           | comma-separated category IDs (empty = all); json: applicable_to_categories |
| applicable_to_products    | text           | comma-separated product IDs (empty = all); json: applicable_to_products |
| free_shipping             | bool           | default false; json: free_shipping                             |
| stackable                 | bool           | default false; json: stackable                                 |
| auto_apply                | bool           | default false; json: auto_apply                                |
| priority                  | int            | default 0; json: priority                                      |
| created_at                | timestamp      | json: created_at                                               |
| updated_at                | timestamp      | json: updated_at                                               |
| deleted_at                | timestamp      | Soft delete; json: deleted_at (omitempty)                      |

### Table: `coupon_usages`
| Column           | Type          | Notes                                     |
|------------------|---------------|-------------------------------------------|
| id               | uint (PK, AI) |                                           |
| coupon_id        | uint (FK)     | FK → coupons.id                           |
| customer_id      | *uint (FK)    | FK → customers.id; nullable               |
| sell_id          | *uint (FK)    | FK → sells.id; nullable                   |
| coupon_code      | string        | snapshot of code at time of use           |
| discount_applied | float64       | actual discount amount applied            |
| original_amount  | float64       | order amount before discount              |
| final_amount     | float64       | order amount after discount               |
| used_at          | timestamp     |                                           |
| created_at       | timestamp     |                                           |

---

## Relationships
- Coupon ← CouponUsages (HasMany)
- CouponUsage → Customer (BelongsTo, optional)
- CouponUsage → Sell (BelongsTo, optional)

---

## Business Logic
- Coupon `code` is unique per company.
- `end_date` must be after `start_date` (validated on create/update).
- On **delete**: the associated image file is removed from disk.
- **GetCouponByCode**: uses a service layer; returns 404 if coupon is not found or inactive.
- **ValidateCoupon**: delegates to `services.ValidateCoupon` which checks:
  - Coupon exists and is active
  - Current date is within start_date/end_date
  - Order amount meets min_order_amount
  - Usage limit not exceeded
  - Per-user usage limit not exceeded
  - Applicability to specific products/categories
  - Returns calculated discount amount on success
- **GetCouponUsageStats**: delegates to `services.GetCouponUsageStats`.
- Scoped by `company_id` from JWT context (except GetCouponByCode which is public).

---

## DTO Validation (CreateCouponRequest)
| Field             | Rules                                                    |
|-------------------|----------------------------------------------------------|
| campaign_name     | required; min=3; max=200                                 |
| code              | required; min=3; max=50; alphanum                        |
| discount          | required; gt=0                                           |
| type              | required; oneof=percentage fixed free_shipping           |
| start_date        | required                                                 |
| end_date          | required; must be after start_date                       |
| status            | optional bool                                            |
| usage_limit       | optional *int                                            |
| usage_limit_per_user | optional *int                                         |
| min_order_amount  | optional float64                                         |
| max_discount      | optional *float64                                        |
| applicable_to_categories | optional string                                   |
| applicable_to_products   | optional string                                   |
| free_shipping     | optional bool                                            |
| stackable         | optional bool                                            |
| auto_apply        | optional bool                                            |
| priority          | optional int                                             |

---

## Endpoints

### GET /coupons/
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Coupons retrieved successfully",
  "data": [ /* array of coupon objects */ ]
}
```

---

### GET /coupons/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Coupon fetched successfully",
  "data": { /* coupon object */ }
}
```

**Error:** `404` — `{"error": "Coupon not found"}`

---

### GET /coupons/code/:code
**Auth:** None (public)
**Purpose:** Look up a coupon by code for storefront use.

**Response 200:**
```json
{
  "message": "Coupon found",
  "data": { /* coupon object */ }
}
```

**Error:** `404` — `{"error": "Coupon not found or inactive"}`

---

### POST /coupons/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
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

**Response 201:**
```json
{
  "message": "Coupon created successfully",
  "data": { /* coupon object */ }
}
```

**Error Responses:**
- `400` — `{"error": "End date must be after start date"}`
- `422` — validation errors: `{"message": "Validation failed", "errors": [...]}`
- `500` — `{"error": "Failed to create coupon: ..."}`

---

### POST /coupons/with-image
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`
**Purpose:** Create a coupon with an optional image file.

**Form Fields:** Same fields as POST /coupons/ (all form-data), plus:
- `start_date` — RFC3339 string (e.g. "2024-06-01T00:00:00Z")
- `end_date` — RFC3339 string
- `image` — optional file (saved to uploads/coupons/)

**Response 201:** Same as POST /coupons/

**Error Responses:**
- `400` — `{"error": "Invalid start_date format (use RFC3339)"}` / `{"error": "Invalid end_date format (use RFC3339)"}`
- `500` — `{"error": "Failed to save image"}`

---

### PUT /coupons/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Partial update — all fields optional.

**Request Body:** Any subset of CreateCouponRequest fields (using UpdateCouponRequest with pointer fields).

**Response 200:**
```json
{
  "message": "Coupon updated successfully",
  "data": { /* coupon object */ }
}
```

**Error Responses:**
- `400` — `{"error": "End date must be after start date"}`
- `404` — `{"error": "Coupon not found"}`
- `422` — validation errors
- `500` — `{"error": "..."}`

---

### PUT /coupons/:id/with-image
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`
**Purpose:** Update coupon with optional image replacement. Old image is deleted if replaced.

**Form Fields:** Any subset of: campaign_name, code, type, status, start_date (RFC3339), end_date (RFC3339), image (file).

**Response 200:**
```json
{
  "message": "Coupon updated successfully",
  "data": { /* partial update map of changed fields */ }
}
```

**Error Responses:**
- `400` — `{"error": "No fields to update"}`
- `404` — `{"error": "Coupon not found"}`
- `500` — `{"error": "Image upload failed"}`

---

### DELETE /coupons/:id
**Auth:** Bearer JWT required
**Purpose:** Soft-delete coupon and remove image file.

**Response 200:**
```json
{
  "message": "Coupon deleted successfully"
}
```

**Error Responses:**
- `404` — `{"error": "Coupon not found"}`
- `500` — `{"error": "Failed to delete coupon"}`

---

### POST /coupons/validate
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Validate a coupon code at checkout.

**Request Body:**
```json
{
  "code": "SUMMER20",
  "customerId": 5,
  "orderAmount": 120.00,
  "cartItems": [
    { "product_id": 2, "category_id": 1, "price": 60.00, "quantity": 2 }
  ]
}
```

**Validation:**
- `code` — required
- `orderAmount` — required; gt=0
- `cartItems[].product_id` — required
- `cartItems[].price` — required; gt=0
- `cartItems[].quantity` — required; gt=0

**Response 200 (valid):** Full validation result from service (includes `valid: true`, calculated `discountAmount`, etc.)

**Response 422 (invalid):**
```json
{
  "valid": false,
  "error_code": "COUPON_EXPIRED",
  "message": "This coupon has expired"
}
```

**Error Responses:**
- `422` — validation errors
- `500` — `{"error": "Validation failed: ..."}`

---

### GET /coupons/:id/usage-stats
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Usage statistics retrieved successfully",
  "data": { /* usage stats from service */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid coupon ID"}`
- `404` — `{"error": "Coupon not found"}`

---

## Dependencies
- **customers** — coupon_usages reference customers
- **orders** — coupon_usages reference sells
- **categories** — coupon applicability by category
- **products** — coupon applicability by product
