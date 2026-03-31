# Frontend Module: Locations (Warehouses / Branches)

## Pages
- `/dashboard/operations/locations` — Location list management
- `/dashboard/operations/locations/new` — Create location form
- `/dashboard/operations/locations/:id` — Location detail / edit

## API Service
`lib/locationApi.ts`

---

## GET /locations
**Purpose:** Fetch all locations (warehouses, stores, branches) for the company
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)

**Note:** No pagination or search parameters — this endpoint returns all locations in a single response.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "name": "Main Warehouse",
      "address": "123 Logistics Blvd, Chicago, IL 60601",
      "contact_person": "Bob Johnson",
      "is_default": true,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "deleted_at": null
    }
  ]
}
```

**Note:** All field names are snake_case (unlike most other modules which use camelCase). Response has no pagination — all locations returned in one call.

**Frontend Impact:**
- Location dropdown in product creation form (`location_id` field)
- Inventory filter by warehouse uses `id`
- Stock transfer source/destination dropdowns
- `is_default` marks the primary location — shown first or pre-selected in dropdowns

---

## GET /locations/:id
**Purpose:** Fetch a single location by ID
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric location ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "name": "Main Warehouse",
    "address": "string",
    "contact_person": "string",
    "is_default": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z",
    "deleted_at": null
  }
}
```

**Frontend Impact:**
- Location edit form pre-populated

---

## POST /locations
**Purpose:** Create a new location
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "Downtown Store",
  "address": "789 Retail Ave, New York, NY 10001",
  "contact_person": "Alice Manager",
  "is_default": false
}
```

**Note:** `name`, `address`, and `contact_person` are required. `is_default` optional (defaults to `false`). If `is_default: true` is set, backend may unset the previous default location.

**Expected Response 201:**
```json
{
  "message": "Location created successfully",
  "data": {
    "id": 2,
    "name": "Downtown Store",
    "address": "789 Retail Ave, New York, NY 10001",
    "contact_person": "Alice Manager",
    "is_default": false,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z",
    "deleted_at": null
  }
}
```

**Frontend Impact:**
- New location appears in all location dropdowns (products, inventory, transfers)
- First location created is typically set as default automatically by backend

---

## PUT /locations/:id
**Purpose:** Update an existing location
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric location ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "string",
  "address": "string",
  "contact_person": "string",
  "is_default": true
}
```

**Note:** All fields optional for partial update.

**Expected Response 200:**
```json
{
  "message": "Location updated successfully",
  "data": { "...full LocationResponse object..." }
}
```

**Frontend Impact:**
- Setting `is_default: true` on one location should clear the flag from all others (backend-enforced)

---

## DELETE /locations/:id
**Purpose:** Soft-delete a location
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric location ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Location deleted successfully"
}
```

**Frontend Impact:**
- Location removed from all dropdowns
- Products and inventory records referencing this location retain the `location_id` but location name may not resolve
- Cannot delete the default location without assigning a new default first (client-side guard recommended)
- Backend may reject deletion if location has active inventory or pending transfers
