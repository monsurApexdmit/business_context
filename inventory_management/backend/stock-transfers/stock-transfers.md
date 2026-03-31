# Module: Stock Transfers

## Business Purpose
Manages the movement of inventory between warehouse locations. Supports transferring both simple product stock and variant-specific stock. Transfers are executed immediately (status = Completed on create). Completed transfers can be cancelled (which reverses the stock movement). Also provides a helper endpoint to list products available at a specific location.

---

## Database Table: `stock_transfers`

| Column           | Type          | Notes                                                     |
|------------------|---------------|-----------------------------------------------------------|
| id               | uint (PK, AI) | json: id                                                  |
| company_id       | uint          | not null; json: companyId                                 |
| product_id       | uint (FK)     | FK → products.id; not null; json: productId               |
| variant_id       | *uint (FK)    | FK → product_variants.id; nullable; json: variantId       |
| from_location_id | uint (FK)     | FK → locations.id; not null; json: fromLocationId         |
| to_location_id   | uint (FK)     | FK → locations.id; not null; json: toLocationId           |
| quantity         | int           | not null; json: quantity                                  |
| status           | enum          | 'Pending', 'Completed', 'Cancelled'; default 'Pending'; json: status |
| notes            | text          | json: notes                                               |
| created_at       | timestamp     | json: createdAt; ordered DESC by default                  |
| updated_at       | timestamp     | json: updatedAt                                           |

> Note: No soft delete on this table.

---

## Relationships
- StockTransfer → Product (BelongsTo)
- StockTransfer → ProductVariant (BelongsTo, optional)
- StockTransfer → FromLocation (BelongsTo via from_location_id)
- StockTransfer → ToLocation (BelongsTo via to_location_id)

---

## Business Logic

### CreateTransfer (Variant)
1. Validate source and destination are different
2. Look up `variant_inventory` row for `(variant_id, from_location_id)`
3. If no row found, fall back to `product_variants.stock` if the product's `location_id` matches — seed a new inventory row
4. Check sufficient quantity; return 400 if insufficient
5. Deduct from source inventory row
6. Upsert destination inventory row (create if not exists, otherwise add quantity)
7. Sync `product_variants.stock` = SUM of all `variant_inventory.quantity` for that variant
8. Create transfer record with status `"Completed"`

### CreateTransfer (Simple Product)
1. Verify product's `location_id` matches `from_location_id`
2. Check sufficient stock; return 400 if insufficient
3. Deduct from `products.stock`
4. If remaining stock = 0, update `products.location_id` to `to_location_id` and set stock = quantity transferred
5. Create transfer record with status `"Completed"`

### CancelTransfer
- Only `Completed` transfers can be cancelled
- Reverses the stock movement (add back to source, deduct from destination)
- Updates status to `"Cancelled"`

---

## Endpoints

### GET /transfers/products-by-location/:location_id
**Auth:** Bearer JWT required
**Purpose:** List products with stock at a specific warehouse. Variants filtered to only those with inventory at the given location.

**Path Params:** `location_id` (uint)

**Query Parameters:**
| Param  | Type   | Default | Description                            |
|--------|--------|---------|----------------------------------------|
| page   | int    | 1       |                                        |
| limit  | int    | 50      | max 100                                |
| search | string |         | LIKE match on product name or SKU      |

**Response 200:**
```json
{
  "message": "Products at location retrieved successfully",
  "location_id": 1,
  "data": [ /* array of product objects with variants filtered to this location */ ],
  "total": 10,
  "page": 1,
  "limit": 50
}
```

**Error Responses:**
- `400` — `{"error": "Invalid location_id"}`
- `401` — `{"error": "Company ID not found in context"}`
- `500` — `{"error": "Failed to retrieve products"}`

---

### GET /transfers/
**Auth:** Bearer JWT required
**Purpose:** List all transfers with optional filtering.

**Query Parameters:**
| Param            | Type   | Default | Description                         |
|------------------|--------|---------|-------------------------------------|
| page             | int    | 1       |                                     |
| limit            | int    | 10      | max 100                             |
| status           | string |         | Pending / Completed / Cancelled; "all" = no filter |
| product_id       | uint   |         | Filter by product                   |
| from_location_id | uint   |         | Filter by source location           |
| to_location_id   | uint   |         | Filter by destination location      |

**Response 200:**
```json
{
  "message": "Transfers retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "productId": 2,
      "product": { "id": 2, "name": "T-Shirt", ... },
      "variantId": 5,
      "variant": { "id": 5, "name": "Small / Red", ... },
      "fromLocationId": 1,
      "fromLocation": { "id": 1, "name": "Main Warehouse", ... },
      "toLocationId": 2,
      "toLocation": { "id": 2, "name": "Branch Store", ... },
      "quantity": 10,
      "status": "Completed",
      "notes": "",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 10
}
```

**Error Responses:**
- `401` — `{"error": "Company ID not found in context"}`
- `500` — `{"error": "Failed to retrieve transfers"}`

---

### POST /transfers/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Execute a stock transfer between locations.

**Request Body:**
```json
{
  "productId": 2,
  "variantId": 5,
  "fromLocationId": 1,
  "toLocationId": 2,
  "quantity": 10,
  "notes": "Moving excess stock"
}
```

**Validation:**
- `productId` — required
- `fromLocationId` — required
- `toLocationId` — required; must differ from fromLocationId
- `quantity` — required; min 1
- `variantId` — optional (omit for simple products)

**Response 201:**
```json
{
  "message": "Transfer completed successfully",
  "data": { /* transfer object with all relations preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `400` — `{"error": "Source and destination warehouse must be different"}`
- `400` — `{"error": "Insufficient stock in source warehouse"}`
- `401` — `{"error": "Company ID not found in context"}`
- `404` — `{"error": "Product not found"}`
- `500` — `{"error": "Failed to create transfer", "details": "..."}`

---

### PUT /transfers/:id/cancel
**Auth:** Bearer JWT required
**Purpose:** Cancel a completed transfer and reverse the stock movement.

**Response 200:**
```json
{
  "message": "Transfer cancelled successfully",
  "data": { /* transfer object with status: "Cancelled" */ }
}
```

**Error Responses:**
- `400` — `{"error": "Only completed transfers can be cancelled"}`
- `401` — `{"error": "Company ID not found in context"}`
- `404` — `{"error": "Transfer not found"}`
- `500` — `{"error": "Failed to cancel transfer", "details": "..."}`

---

## Dependencies
- **products** — product stock is deducted/restored
- **locations** — from and to locations must exist
- **inventory** — variant_inventory rows are updated on transfer
