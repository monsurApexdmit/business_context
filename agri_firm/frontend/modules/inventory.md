# Farm Supplies & Equipment Inventory Module

**Frontend Route:** `/inventory`
**API Base Path:** `/inventory`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interface

```typescript
interface InventoryItem {
  id: string;
  name: string;
  category: "Seeds" | "Fertilizer" | "Equipment" | "Feed" | "Medicine" | "Tools" | "Other";
  quantity: number;                                                      // Current stock level
  unit: "kg" | "liters" | "bags" | "pieces" | "boxes" | "tons" | "bottles";
  minStock: number;                                                      // Minimum threshold
  costPerUnit: number;                                                   // In local currency
  supplier: string;
  location: string;                                                      // Storage location e.g. "Warehouse A"
  lastRestocked: string;                                                 // ISO date string
}

// Derived client-side — NOT stored in backend:
type StockStatus = "in-stock" | "low-stock" | "out-of-stock";

// Logic:
// quantity === 0           → "out-of-stock"
// quantity <= minStock     → "low-stock"
// quantity > minStock      → "in-stock"
```

---

## Stock Status — Client-Side Derivation

**Important:** The `stockStatus` field is **derived client-side** from `quantity` and `minStock`. The backend does NOT store a `stockStatus` column.

However, the backend **must support filtering by stockStatus** via a query parameter. To implement this, the backend should:

- `stockStatus=in-stock` → return items where `quantity > minStock`
- `stockStatus=low-stock` → return items where `quantity > 0 AND quantity <= minStock`
- `stockStatus=out-of-stock` → return items where `quantity = 0`

This logic must be implemented as a SQL `WHERE` condition, not post-filtered in application code, to ensure correct pagination counts.

---

## Endpoints

---

### GET `/inventory`

List all inventory items with optional filters and pagination.

**Auth Required:** Yes

**Query Parameters:**

| Param         | Type    | Required | Description                                                     |
|---------------|---------|----------|-----------------------------------------------------------------|
| `farm_id`     | string  | Yes      | Active farm UUID                                                |
| `search`      | string  | No       | Partial match on `name` or `supplier`                           |
| `category`    | string  | No       | Filter by category: `Seeds\|Fertilizer\|Equipment\|Feed\|Medicine\|Tools\|Other` |
| `stockStatus` | string  | No       | Filter by derived stock status: `in-stock\|low-stock\|out-of-stock` |
| `page`        | integer | No       | Page number, default `1`                                        |
| `limit`       | integer | No       | Page size, default `20`                                         |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 67,
    "page": 1,
    "limit": 20,
    "data": [
      {
        "id": "item-uuid-001",
        "name": "NPK Fertilizer",
        "category": "Fertilizer",
        "quantity": 150,
        "unit": "bags",
        "minStock": 50,
        "costPerUnit": 25.00,
        "supplier": "AgriSupply Co.",
        "location": "Warehouse B",
        "lastRestocked": "2024-03-01"
      }
    ]
  }
}
```

**Frontend Impact:**
- The table derives and displays `stockStatus` badge from `quantity` and `minStock` per item.
- Total value (shown in stats) is computed client-side as `sum(quantity * costPerUnit)` — no endpoint needed for this calculation, but the stats endpoint returns the pre-computed value.

---

### POST `/inventory`

Create a new inventory item.

**Auth Required:** Yes

**Request Body:**

```json
{
  "name": "NPK Fertilizer",
  "category": "Fertilizer",
  "quantity": 150,
  "unit": "bags",
  "minStock": 50,
  "costPerUnit": 25.00,
  "supplier": "AgriSupply Co.",
  "location": "Warehouse B",
  "lastRestocked": "2024-03-01",
  "farm_id": "farm-uuid-here"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Inventory item created successfully",
  "data": { /* full InventoryItem object with id */ }
}
```

**Validation Rules:**
- `name`: required, 1–100 characters
- `category`: required, must be one of the enum values
- `quantity`: required, integer >= 0
- `unit`: required, must be one of the enum values
- `minStock`: required, integer >= 0
- `costPerUnit`: required, number >= 0, max 2 decimal places
- `supplier`: optional string, max 100 characters
- `location`: optional string, max 100 characters
- `lastRestocked`: optional, valid ISO date
- `farm_id`: required

---

### GET `/inventory/stats`

Returns aggregate statistics for inventory.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description      |
|-----------|--------|----------|------------------|
| `farm_id` | string | Yes      | Active farm UUID |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "totalItems": 67,
    "totalValue": 48250.00,
    "lowStockCount": 8,
    "outOfStockCount": 3
  }
}
```

**Field Definitions:**
- `totalItems`: count of all inventory items for the farm
- `totalValue`: `SUM(quantity * costPerUnit)` across all items
- `lowStockCount`: count where `quantity > 0 AND quantity <= minStock`
- `outOfStockCount`: count where `quantity = 0`

**Important — Routing Note:** Register `/inventory/stats` **before** `/inventory/:id` to prevent routing conflict.

---

### GET `/inventory/:id`

Fetch a single inventory item by ID.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "item-uuid-001",
    "name": "NPK Fertilizer",
    "category": "Fertilizer",
    "quantity": 150,
    "unit": "bags",
    "minStock": 50,
    "costPerUnit": 25.00,
    "supplier": "AgriSupply Co.",
    "location": "Warehouse B",
    "lastRestocked": "2024-03-01"
  }
}
```

**Error Response:**

| Code  | Scenario    |
|-------|-------------|
| `404` | Item not found |

---

### PUT `/inventory/:id`

Full update of an inventory item.

**Auth Required:** Yes

**Request Body:** Same shape as `POST /inventory` (all fields, without `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Inventory item updated successfully",
  "data": { /* updated InventoryItem object */ }
}
```

**Note:** This endpoint is also used to restock an item — simply update the `quantity` and `lastRestocked` date fields.

---

### DELETE `/inventory/:id`

Delete an inventory item.

**Auth Required:** Yes

**Success Response — `204 No Content`**

---

## Notes

- `costPerUnit` should be stored as a decimal number with 2 decimal places. Return as a JSON `number`, not a string.
- The frontend displays a total inventory value computed as `SUM(quantity * costPerUnit)`. This is also pre-computed and returned by `/inventory/stats` as `totalValue` for display in stat cards.
- `unit` is a controlled enum. If the user needs a custom unit not in the list, the `Other` category can be used as a workaround until a custom unit field is added.
- `lastRestocked` is informational only — used for display in the item details view. No automated restocking logic is expected in the current system.
- Consider adding an audit log or restocking history endpoint in a future iteration (e.g. `GET /inventory/:id/history`).
- Low-stock and out-of-stock items may trigger notifications via the Notifications module. The backend should check stock levels after each `PUT /inventory/:id` and create notification records if thresholds are breached.
