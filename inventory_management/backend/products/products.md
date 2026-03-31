# Module: Products

## Business Purpose
Manages the product catalog. Products can be simple (with a single stock) or have variants (size, color, etc. with individual stock per warehouse). Products belong to a company and can be associated with categories, vendors, locations, and attributes. Supports file uploads for main product image and a gallery. Stock is managed through the products table (simple products) or variant_inventory table (variant products).

---

## Database Tables

### Table: `products`
| Column         | Type           | Notes                                                   |
|----------------|----------------|---------------------------------------------------------|
| id             | uint (PK, AI)  |                                                         |
| company_id     | uint (FK)      | FK → companies.id; indexed                              |
| name           | varchar(255)   | not null                                                |
| description    | text           |                                                         |
| category_id    | *uint (FK)     | FK → categories.id; nullable                            |
| vendor_id      | *uint (FK)     | FK → users.id; nullable                                 |
| location_id    | *uint (FK)     | FK → locations.id; nullable                             |
| price          | float64        | default 0                                               |
| sale_price     | float64        | default 0                                               |
| cost_price     | float64        | default 0; purchase/acquisition cost from vendor        |
| stock          | int            | default 0; synced from variants if variants exist       |
| sku            | varchar(100)   | unique per company: idx_product_company_sku             |
| barcode        | varchar(100)   | unique per company: idx_product_company_barcode         |
| published      | bool           | default false                                           |
| receipt_number | string         |                                                         |
| image          | string         | path to main image file                                 |
| created_at     | timestamp      | Auto                                                    |
| updated_at     | timestamp      | Auto                                                    |
| deleted_at     | timestamp      | Soft delete index                                       |

### Table: `product_variants`
| Column     | Type          | Notes                                        |
|------------|---------------|----------------------------------------------|
| id         | uint (PK)     |                                              |
| product_id | uint (FK)     | FK → products.id; unique with sku            |
| name       | varchar(255)  | e.g. "Small / Red"                           |
| attributes | JSON          | {"Size": "Small", "Color": "Red"}            |
| price      | float64       | default 0                                    |
| sale_price | float64       | default 0                                    |
| cost_price | float64       | default 0; overrides product cost_price if set |
| stock      | int           | synced from sum of variant_inventory.quantity|
| sku        | varchar(100)  | unique: idx_product_sku (product_id + sku)   |
| barcode    | varchar(100)  | indexed                                      |
| created_at | timestamp     |                                              |
| updated_at | timestamp     |                                              |
| deleted_at | timestamp     | Soft delete                                  |

### Table: `product_images`
| Column     | Type      | Notes                              |
|------------|-----------|------------------------------------|
| id         | uint (PK) |                                    |
| product_id | uint (FK) | FK → products.id; CASCADE delete   |
| path       | string    | file path                          |
| position   | int       | default 0; ordering                |
| is_primary | bool      | default false                      |
| created_at | timestamp |                                    |

### Table: `variant_inventory`
| Column      | Type      | Notes                            |
|-------------|-----------|----------------------------------|
| id          | uint (PK) |                                  |
| variant_id  | uint (FK) | FK → product_variants.id; CASCADE|
| location_id | uint (FK) | FK → locations.id                |
| quantity    | int       | default 0                        |
| created_at  | timestamp |                                  |
| updated_at  | timestamp |                                  |

### Join Table: `product_attributes` (many-to-many)
| Column       | Notes                |
|--------------|----------------------|
| product_id   | FK → products.id     |
| attribute_id | FK → attributes.id   |

---

## Relationships
- Product → Category (BelongsTo, optional)
- Product → User/Vendor (BelongsTo via vendor_id, optional)
- Product → Location (BelongsTo, optional)
- Product ↔ Attributes (ManyToMany via product_attributes)
- Product → ProductVariants (HasMany, CASCADE delete)
- Product → ProductImages (HasMany, CASCADE delete)
- ProductVariant → VariantInventory (HasMany via variant_id, CASCADE delete)
- VariantInventory → Location (BelongsTo)

---

## Business Logic

### Stock Management
- **Simple products** (no variants): `products.stock` holds the total
- **Products with variants**: `product_variants.stock` = sum of all `variant_inventory.quantity` for that variant. `products.stock` = sum of all variant stocks (synced automatically on every stock change)
- On order creation: stock is deducted atomically from variant_inventory or products.stock
- On order deletion: stock is restored
- On stock transfer: inventory moved between locations, stocks synced

### Product Creation (multipart/form-data)
1. Parse form fields and optional image/gallery files
2. Begin DB transaction
3. Create product record (gets ID)
4. Create variants with correct product_id
5. Seed variant_inventory records
6. Two-phase file commit: save images to temp, then move to final path after DB success
7. Rollback temp files on DB failure

### Image Upload
- Main image field: `image` (single file)
- Gallery field: `images[]` (multiple files)
- Files stored under `./uploads/products/`
- Image paths saved in `products.image` and `product_images.path`

---

## Endpoints

### GET /products/
**Auth:** Bearer JWT required
**Purpose:** List products for authenticated company with pagination and filters.

**Query Parameters:**
| Param       | Type   | Description                     |
|-------------|--------|---------------------------------|
| page        | int    | default 1                       |
| limit       | int    | default 10                      |
| category_id | uint   | filter by category               |
| vendor_id   | uint   | filter by vendor                 |
| location_id | uint   | filter by location               |

**Response 200:**
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
      "attributes": [...],
      "variants": [...],
      "images": [...],
      "category": {...},
      "vendor": {...},
      "location": {...},
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

---

### GET /products/:id
**Auth:** Bearer JWT required
**Path Params:** `id` (uint)

**Response 200:**
```json
{
  "message": "Product fetched successfully",
  "data": { /* full product object with all relations */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid product ID"}`
- `404` — `{"error": "Product not found"}`
- `500` — `{"error": "Failed to fetch product"}`

---

### POST /products/
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`
**Purpose:** Create a new product.

**Form Fields:**
| Field        | Type    | Required | Notes                              |
|--------------|---------|----------|------------------------------------|
| name         | string  | yes      |                                    |
| description  | string  | no       |                                    |
| category_id  | uint    | no       |                                    |
| vendor_id    | uint    | no       |                                    |
| location_id  | uint    | no       |                                    |
| price        | float   | no       |                                    |
| sale_price   | float   | no       |                                    |
| cost_price   | float   | no       | Purchase/acquisition cost          |
| stock        | int     | no       |                                    |
| sku          | string  | no       |                                    |
| barcode      | string  | no       |                                    |
| published    | bool    | no       |                                    |
| receipt_number | string | no      |                                    |
| attributes   | string  | no       | JSON string: `[1, 2, 3]` (IDs)     |
| variants     | string  | no       | JSON string of variant objects     |
| image        | file    | no       | Main product image                 |
| images[]     | file[]  | no       | Gallery images                     |

**Variants JSON format:**
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

**Response 201:**
```json
{
  "message": "Product created successfully",
  "data": { /* full product object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}` / `{"error": "product name is required"}`
- `422` — `{"error": "Invalid reference: vendor, category, or location ID does not exist"}`
- `500` — `{"error": "Failed to create product", "details": "..."}`

---

### PUT /products/:id
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`
**Purpose:** Update product (partial update — only provided fields are changed).

**Form Fields:** Same as POST (all optional). New images replace existing ones.

**Response 200:**
```json
{
  "message": "Product updated successfully",
  "data": { /* full product object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid product ID"}`
- `404` — `{"error": "Product not found"}`
- `422` — `{"error": "Invalid reference: vendor, category, or location ID does not exist"}`
- `500` — `{"error": "Failed to update product", "details": "..."}`

---

### PATCH /products/:id/status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Toggle published status.

**Request Body:**
```json
{
  "published": true
}
```

**Response 200:**
```json
{
  "message": "Product status updated successfully",
  "data": { /* full product object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid product ID"}` / `{"error": "Invalid request body", "details": "..."}`
- `404` — `{"error": "Product not found"}`

---

### DELETE /products/:id
**Auth:** Bearer JWT required
**Purpose:** Soft-delete product and clean up image files.

**Response 200:**
```json
{
  "message": "Product deleted successfully"
}
```

**Error Responses:**
- `400` — `{"error": "Invalid product ID"}`
- `404` — `{"error": "Product not found"}`
- `500` — `{"error": "Failed to delete product"}`

---

## Dependencies
- **categories** — product belongs to a category
- **locations** — product and variant inventory scoped to a location
- **users (vendors)** — product can reference a vendor via vendor_id
- **attributes** — many-to-many association

---

## Important Notes
- All queries are scoped by `company_id` from JWT
- Product deletion is soft (sets `deleted_at`); files are cleaned up
- `stock` on product is auto-synced from variant stocks when variants are present
- Unique constraints on `(company_id, sku)` and `(company_id, barcode)`
