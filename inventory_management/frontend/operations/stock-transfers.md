# Frontend Module: Stock Transfers

## Pages
- `/dashboard/operations/transfers` — Transfer list (all inter-location transfers)
- `/dashboard/operations/transfers/new` — Create new stock transfer
- `/dashboard/operations/transfers/:id` — Transfer detail with cancel option

## API Service
`lib/transferApi.ts`

---

## GET /transfers
**Purpose:** Fetch all stock transfer records for the company
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)

**Note:** No pagination or filter parameters in the current implementation — all transfers returned in a single response. `page`, `limit`, `total` are present in the response type but not sent as query params.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "productId": 10,
      "product": { "id": 10, "name": "Blue T-Shirt" },
      "variantId": 2,
      "variant": { "id": 2, "name": "Blue / Large" },
      "fromLocationId": 1,
      "fromLocation": { "id": 1, "name": "Main Warehouse" },
      "toLocationId": 2,
      "toLocation": { "id": 2, "name": "Downtown Store" },
      "quantity": 20,
      "notes": "Seasonal restock",
      "status": "Pending | Completed | Cancelled",
      "createdAt": "2024-01-10T00:00:00Z",
      "updatedAt": "2024-01-10T00:00:00Z"
    }
  ],
  "page": 1,
  "limit": 50,
  "total": 25
}
```

**Note:** Pagination fields `page`, `limit`, `total` are at top level of response. `variant` may be null if the transferred item is a base product (not a variant).

**Frontend Impact:**
- Transfer list table showing source/destination locations, product, quantity, status
- Status badges: Pending (yellow), Completed (green), Cancelled (red)
- Cancel button visible only for Pending transfers

---

## GET /transfers/products-by-location/:locationId
**Purpose:** Fetch all products and variants available at a specific location (for transfer source selection)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `locationId` — numeric location ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 10,
      "name": "Blue T-Shirt",
      "stock": 50,
      "sku": "TSH-BL",
      "variants": [
        {
          "id": 2,
          "product_id": 10,
          "name": "Blue / Large",
          "stock": 20,
          "sku": "TSH-BL-LG"
        },
        {
          "id": 3,
          "product_id": 10,
          "name": "Blue / Medium",
          "stock": 30,
          "sku": "TSH-BL-MD"
        }
      ]
    }
  ]
}
```

**Note:** Response has no standard `{ message, data }` wrapper in TypeScript type — it's typed as `LocationProductsResponse` with `{ message: string; data: LocationProductRaw[] }`.

The frontend UI flattens this nested structure into `LocationProduct` rows:
```typescript
interface LocationProduct {
  type: 'variant' | 'product';
  id: number;         // variant.id if type === 'variant', else product.id
  productId: number;
  productName: string;
  variantName?: string;
  sku: string;
  stock: number;
}
```

Products with variants are flattened to one row per variant. Products without variants create one `type: "product"` row.

**Frontend Impact:**
- Transfer creation form: when user selects "From Location", this endpoint populates the product/variant dropdown
- Shows current stock at source location to validate transfer quantity
- Only items with `stock > 0` should be selectable (client-side validation)

---

## POST /transfers
**Purpose:** Create a new stock transfer between locations
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "productId": 10,
  "variantId": 2,
  "fromLocationId": 1,
  "toLocationId": 2,
  "quantity": 20,
  "notes": "Seasonal restock for downtown store"
}
```

**Note:** `productId`, `fromLocationId`, `toLocationId`, and `quantity` are required. `variantId` optional (omit for base product transfers). `notes` optional. `fromLocationId` and `toLocationId` must be different.

**Expected Response 201:**
```json
{
  "message": "Transfer created successfully",
  "data": {
    "id": 5,
    "productId": 10,
    "product": { "id": 10, "name": "Blue T-Shirt" },
    "variantId": 2,
    "variant": { "id": 2, "name": "Blue / Large" },
    "fromLocationId": 1,
    "fromLocation": { "id": 1, "name": "Main Warehouse" },
    "toLocationId": 2,
    "toLocation": { "id": 2, "name": "Downtown Store" },
    "quantity": 20,
    "notes": "Seasonal restock for downtown store",
    "status": "Pending",
    "createdAt": "2024-01-10T00:00:00Z",
    "updatedAt": "2024-01-10T00:00:00Z"
  }
}
```

**Note:** Transfer is created with `status: "Pending"`. The backend may immediately process the inventory movement (set to "Completed") or require a separate confirmation step — verify backend behavior.

**Frontend Impact:**
- Transfer created and appears in list
- If processed immediately, source location stock decremented and destination incremented
- Transfer list refreshed after creation

---

## PUT /transfers/:id/cancel
**Purpose:** Cancel a pending stock transfer
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric transfer ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:** None (PUT with no body)

**Note:** Uses `PUT` method (not `PATCH` or `DELETE`) for the cancel action. No request body needed.

**Expected Response 200:**
```json
{
  "message": "Transfer cancelled successfully",
  "data": { "...TransferResponse with status: 'Cancelled'..." }
}
```

**Frontend Impact:**
- Cancel button on transfer detail page (only shown for Pending transfers)
- If the transfer had already moved stock, backend may reverse the inventory adjustment
- Cancelled transfers remain in history for audit purposes
