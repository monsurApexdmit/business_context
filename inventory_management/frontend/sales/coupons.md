# Frontend Module: Coupons

## Pages
- `/dashboard/sales/coupons` — Coupon list management
- `/dashboard/sales/coupons/new` — Create coupon form
- `/dashboard/sales/coupons/:id/edit` — Edit coupon form

## API Service
`lib/couponApi.ts`

---

## GET /coupons
**Purpose:** Fetch paginated list of coupons
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by campaign name or code (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "campaign_name": "Summer Sale",
      "code": "SUMMER20",
      "discount": 20.0,
      "type": "percentage | fixed",
      "status": true,
      "image": "string",
      "start_date": "2024-06-01T00:00:00Z",
      "end_date": "2024-08-31T00:00:00Z",
      "usage_limit": 100,
      "usage_limit_per_user": 1,
      "times_used": 35,
      "min_order_amount": 50.00,
      "max_discount": 30.00,
      "applicable_to_categories": "string",
      "applicable_to_products": "string",
      "free_shipping": false,
      "stackable": false,
      "auto_apply": false,
      "priority": 1,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "deleted_at": null
    }
  ]
}
```

**Note:** All fields are in snake_case (backend returns snake_case for this module). `status: true` means active, `status: false` means inactive/expired.

**Frontend Impact:**
- Coupon list table with active/inactive status badges
- `times_used` / `usage_limit` shown as usage bar
- Coupon code displayed as copyable chip

---

## GET /coupons/:id
**Purpose:** Fetch a single coupon by ID
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric coupon ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full CouponResponse object..." }
}
```

**Frontend Impact:**
- Edit form pre-populated
- Image preview shown if `image` URL is present

---

## POST /coupons
**Purpose:** Create a new coupon (JSON only, no image)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "campaign_name": "Summer Sale",
  "code": "SUMMER20",
  "type": "percentage | fixed | free_shipping",
  "discount": 20.0,
  "status": true,
  "start_date": "2024-06-01T00:00:00Z",
  "end_date": "2024-08-31T00:00:00Z",
  "min_order_amount": 50.00,
  "usage_limit": 100
}
```

**Note:** `campaign_name` (min 3 chars), `code` (alphanumeric only), `type`, and `discount` (> 0) are required. Dates must be RFC3339 format. `end_date` must be after `start_date`.

**Expected Response 201:**
```json
{
  "message": "Coupon created successfully",
  "data": { "...full CouponResponse object..." }
}
```

**Frontend Impact:**
- Use this endpoint when creating without an image
- Coupon appears in list immediately after creation

---

## POST /coupons/with-image
**Purpose:** Create a new coupon with a banner/image attachment
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data` (Content-Type header deleted; browser sets boundary automatically)

**Query Params:**
- `company_id` (auto-injected)

**Form Fields:**

| Field | Type | Notes |
|-------|------|-------|
| `campaign_name` | string | Required |
| `code` | string | Required, alphanumeric |
| `type` | string | `"percentage"` / `"fixed"` / `"free_shipping"` |
| `discount` | string (number) | Required |
| `status` | string (`"true"`/`"false"`) | Optional |
| `start_date` | string (RFC3339) | Optional |
| `end_date` | string (RFC3339) | Optional |
| `min_order_amount` | string (number) | Optional |
| `usage_limit` | string (number) | Optional |
| `image` | File | Required for this endpoint |

**Note:** The axios interceptor detects `FormData` and deletes the `Content-Type` header so the browser can set it with the correct multipart boundary. All non-file fields are converted to strings via `String(value)`.

**Expected Response 201:**
```json
{
  "message": "Coupon created successfully",
  "data": { "...full CouponResponse object with image path..." }
}
```

**Frontend Impact:**
- Used when user uploads a coupon banner image
- Image path returned in `data.image`

---

## PUT /coupons/:id
**Purpose:** Update an existing coupon (JSON only, no image)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric coupon ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "campaign_name": "string",
  "code": "string",
  "type": "percentage | fixed | free_shipping",
  "discount": 20.0,
  "status": true,
  "start_date": "2024-06-01T00:00:00Z",
  "end_date": "2024-08-31T00:00:00Z",
  "min_order_amount": 50.00,
  "usage_limit": 100
}
```

**Note:** All fields optional for partial update.

**Expected Response 200:**
```json
{
  "message": "Coupon updated successfully",
  "data": { "...full CouponResponse object..." }
}
```

**Frontend Impact:**
- Edit form updates coupon details without changing image

---

## PUT /coupons/:id/with-image
**Purpose:** Update an existing coupon with a new image
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data` (Content-Type header deleted; browser sets boundary automatically)

**Path Params:**
- `id` — numeric coupon ID

**Query Params:**
- `company_id` (auto-injected)

**Form Fields:** Same as `POST /coupons/with-image` above (all optional except `image`).

**Note:** Same Content-Type deletion behavior as POST with-image. Uses `PUT` method (not PATCH) so the backend replaces the image.

**Expected Response 200:**
```json
{
  "message": "Coupon updated successfully",
  "data": { "...full CouponResponse object with new image path..." }
}
```

**Frontend Impact:**
- Replaces existing coupon image
- Old image may be orphaned on server if not cleaned up by backend

---

## DELETE /coupons/:id
**Purpose:** Delete a coupon
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric coupon ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Coupon deleted successfully"
}
```

**Frontend Impact:**
- Coupon removed from list
- Active orders using this coupon code are unaffected (code stored on order)

---

## POST /coupons/validate
**Purpose:** Validate a coupon code and calculate discount for a given order amount
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "code": "SUMMER20",
  "order_amount": 100.00
}
```

**Note:** The frontend sends `order_amount` (snake_case) — NOT `cartTotal` as mentioned in the original session notes. The TypeScript function parameter is `orderAmount` but the serialized key is `order_amount`.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full CouponResponse object..." },
  "discount": 20.00
}
```

**Note:** Response has THREE top-level fields: `message`, `data` (coupon details), and `discount` (calculated discount amount). This is NOT the standard `{ message, data }` envelope.

**Frontend Impact:**
- Coupon code input in checkout/order creation form
- `discount` value applied to order total display
- Error response (4xx) shown as "Invalid coupon" or "Minimum order not met"

---

## GET /coupons/by-code/:code
**Purpose:** Fetch coupon details by code string
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `code` — coupon code string (e.g., `SUMMER20`)

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full CouponResponse object..." }
}
```

**Frontend Impact:**
- Used to look up coupon details without validating against a cart total
- Can be used to pre-fill coupon edit form when accessed via code
