# Frontend Module: Products

## Pages
- `/dashboard/products` — Product list with search, filter, pagination
- `/dashboard/products/new` — Create product form
- `/dashboard/products/:id` — Edit product form
- `/dashboard/products/:id/view` — Product detail view

## API Service
`lib/productApi.ts`

---

## GET /products
**Purpose:** Fetch paginated product list with optional filters
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by name/SKU (optional)
- `status` — filter by status string (optional)
- `category` — filter by category name (optional)
- `category_id` — filter by category ID (optional)
- `location_id` — filter by warehouse/location ID (optional)
- `vendor_id` — filter by vendor ID (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "name": "string",
      "description": "string",
      "category": "string | { id, category_name, parent_id, status, created_at, updated_at, deleted_at }",
      "location_id": 1,
      "location": {
        "id": 1,
        "name": "string",
        "address": "string",
        "contact_person": "string",
        "is_default": true
      },
      "price": 29.99,
      "sale_price": 24.99,
      "stock": 50,
      "status": "string",
      "published": true,
      "image": "string",
      "images": [
        { "id": 1, "product_id": 1, "path": "string", "position": 0, "is_primary": true }
      ],
      "sku": "string",
      "barcode": "string",
      "vendor_id": 1,
      "receipt_number": "string",
      "attributes": [
        {
          "id": 1,
          "name": "string",
          "display_name": "string",
          "option_type": "string",
          "values": "red,green,blue",
          "description": "string",
          "is_required": false,
          "status": true,
          "sort_order": 0,
          "created_at": "2024-01-01T00:00:00Z",
          "updated_at": "2024-01-01T00:00:00Z",
          "deleted_at": null
        }
      ],
      "variants": [
        {
          "id": 1,
          "product_id": 1,
          "name": "Red / Large",
          "sku": "string",
          "barcode": "string",
          "price": 29.99,
          "sale_price": 24.99,
          "stock": 10,
          "attributes": { "color": "red", "size": "large" }
        }
      ],
      "inventory": [
        { "warehouse_id": 1, "quantity": 50 }
      ],
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 20,
    "total_pages": 5,
    "has_next": true,
    "has_previous": false
  }
}
```

**Note:** `category` field may return either a string (category name) or a full `CategoryResponse` object — frontend must handle both shapes.

**Frontend Impact:**
- Product table rendered from `data` array
- Pagination controls driven by `pagination` object
- Category filter dropdown populated from categories module

---

## GET /products/:id
**Purpose:** Fetch full details for a single product including variants, images, attributes
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric product ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full ProductResponse object (same shape as list items above)..." }
}
```

**Frontend Impact:**
- Edit form pre-populated with all product fields
- Variant table and image gallery built from nested arrays

---

## POST /products/
**Purpose:** Create a new product with optional images and variants
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data` (auto-set by browser when using FormData)

**Query Params:**
- `company_id` (auto-injected)

**Form Fields (built by `buildFormData()`):**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | |
| `description` | string | no | |
| `category_id` | string (number) | yes | |
| `location_id` | string (number) | yes | |
| `price` | string (number) | yes | |
| `sale_price` | string (number) | yes | |
| `stock` | string (number) | yes | |
| `sku` | string | no | Omitted if empty |
| `barcode` | string | no | Omitted if empty |
| `published` | string ("true"/"false") | no | |
| `vendor_id` | string (number) | no | Only appended if not NaN |
| `receipt_number` | string | no | Omitted if empty |
| `attributes` | JSON string | no | Array of attribute IDs only: `[1, 2, 3]` — NOT full attribute objects |
| `variants` | JSON string | no | Full variants array as JSON |
| `image[0]` | File | no | First image file |
| `image[1]` | File | no | Second image file |
| `image[N]` | File | no | Nth image file (indexed) |

**Critical Note on `attributes` field:** The frontend extracts only the integer IDs from the attributes array and sends `JSON.stringify([1, 2, 3])`. It does NOT send the full `{ id, name, value }` objects. The backend must accept an array of attribute IDs.

**Critical Note on images:** Images are sent as `image[0]`, `image[1]`, etc. (bracket-indexed), NOT as a single `images[]` field.

**Expected Response 201:**
```json
{
  "message": "Product created successfully",
  "data": { "...full ProductResponse object..." }
}
```

**Frontend Impact:**
- On success: navigates to product list or product detail page
- Newly created product appears in list
- Image paths returned in `data.images` used for display

---

## PUT /products/:id
**Purpose:** Update an existing product (full replace, but only provided fields are sent)
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`

**Path Params:**
- `id` — numeric product ID

**Query Params:**
- `company_id` (auto-injected)

**Form Fields:** Same as POST `/products/` — all fields optional for update. Same `buildFormData()` function used.

**Expected Response 200:**
```json
{
  "message": "Product updated successfully",
  "data": { "...full ProductResponse object..." }
}
```

**Frontend Impact:**
- Edit form shows success notification
- Product list row updated if currently cached
- New images appended or existing images replaced depending on backend logic

---

## PATCH /products/:id/status
**Purpose:** Update only the status field of a product (e.g., active/inactive)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric product ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "status": "string"
}
```

**Note:** This endpoint explicitly sets `Content-Type: application/json` in the request config, overriding any multipart default. The `status` value is a free-form string (e.g., `"active"`, `"inactive"`, `"draft"`).

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full ProductResponse object..." }
}
```

**Frontend Impact:**
- Status toggle/dropdown in product list row — no page reload needed
- Product status badge updates immediately

---

## DELETE /products/:id
**Purpose:** Soft-delete a product
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric product ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Product deleted successfully"
}
```

**Frontend Impact:**
- Product removed from list
- If product has active orders/inventory, backend may reject with 4xx (frontend should handle)
- Soft delete — product may still appear in order history references
