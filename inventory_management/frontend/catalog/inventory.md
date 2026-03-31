# Frontend Module: Inventory

## Pages
- `/dashboard/inventory` — Inventory levels across all locations/warehouses

## API Service
`lib/inventoryApi.ts`

---

## GET /inventory
**Purpose:** Fetch inventory levels for all products and variants, optionally filtered by location/warehouse
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `search` — search by product name or SKU (optional)
- `warehouse_id` — filter by specific warehouse/location ID (optional)
- `page` — page number (optional)
- `limit` — items per page (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "type": "variant | product",
      "id": 1,
      "productId": 10,
      "productName": "Blue T-Shirt",
      "variantName": "Blue / Large",
      "sku": "TSH-BL-LG",
      "barcode": "1234567890",
      "stock": 25,
      "inventory": [
        {
          "locationId": 1,
          "locationName": "Main Warehouse",
          "quantity": 15
        },
        {
          "locationId": 2,
          "locationName": "Store Front",
          "quantity": 10
        }
      ]
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 150
}
```

**Note:** Response pagination fields (`page`, `limit`, `total`) are at the top level of the response object, NOT nested inside `data`. This differs from other list endpoints that use a nested `pagination` object.

**Note on `type` field:** When `type: "product"`, the item represents the base product stock. When `type: "variant"`, the item represents a specific variant's stock. The `id` field corresponds to the variant ID for variants, or product ID for base products.

**Frontend Impact:**
- Inventory table shows one row per product/variant SKU
- `inventory` array shows per-location breakdown (multi-column or expandable row)
- Low stock threshold warning (client-side check against user-defined threshold)
- `warehouse_id` filter populates from the Locations module (`locationApi`)
- Searching by SKU/name filters across both products and variants
