# Module: Inventory

## Business Purpose
Provides a read-only view of current stock levels across all products and their variants, broken down by warehouse/location. Returns a flat list of inventory rows — one row per simple product, or one row per variant for variant products.

---

## Data Model (No Dedicated Table)
Inventory is a computed view over:
- `products` table (for simple products)
- `product_variants` table (for variant products)
- `variant_inventory` table (per-location quantities for variants)
- `locations` table (for location names)

---

## Response Row Shape

```json
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
}
```

For variant rows:
```json
{
  "type": "variant",
  "id": 5,
  "productId": 1,
  "productName": "T-Shirt",
  "variantName": "Small / Red",
  "sku": "TSHIRT-S-RED",
  "barcode": "",
  "stock": 50,
  "inventory": [
    { "locationId": 1, "locationName": "Main Warehouse", "quantity": 30 },
    { "locationId": 2, "locationName": "Branch Store", "quantity": 20 }
  ]
}
```

---

## Endpoints

### GET /inventory/
**Auth:** Bearer JWT required
**Purpose:** List all inventory rows with per-warehouse breakdown.

**Query Parameters:**
| Param       | Type   | Default | Description                              |
|-------------|--------|---------|------------------------------------------|
| page        | int    | 1       |                                          |
| limit       | int    | 10      | max 100                                  |
| search      | string |         | LIKE match on product name or SKU        |
| location_id | uint   |         | Filter to products at specific location  |

**Response 200:**
```json
{
  "message": "Inventory retrieved successfully",
  "data": [
    {
      "type": "product",
      "id": 1,
      "productId": 1,
      "productName": "T-Shirt",
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
        { "locationId": 1, "locationName": "Main Warehouse", "quantity": 50 }
      ]
    }
  ],
  "total": 25,
  "page": 1,
  "limit": 10
}
```

**Error Responses:**
- `401` — `{"error": "Company ID not found in context"}`
- `500` — `{"error": "Failed to retrieve inventory"}`

---

## Dependencies
- **products** — product catalog
- **locations** — warehouse location names
- **stock-transfers** — transfers update variant_inventory which is reflected here
