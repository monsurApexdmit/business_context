# Module: Locations

## Business Purpose
Manages warehouse/store locations for a company. Locations are used to track inventory per warehouse. Products and variant inventory are scoped to locations. One location can be marked as the default.

---

## Database Table: `locations`

| Column         | Type          | Notes                                      |
|----------------|---------------|--------------------------------------------|
| id             | uint (PK, AI) | json: id                                   |
| company_id     | uint (FK)     | FK → companies.id; indexed; json: companyId|
| name           | string        | json: name                                 |
| address        | string        | json: address                              |
| contact_person | string        | json: contact_person                       |
| is_default     | bool          | default false; json: is_default            |
| created_at     | timestamp     | json: created_at                           |
| updated_at     | timestamp     | json: updated_at                           |
| deleted_at     | timestamp     | Soft delete; json: deleted_at              |

---

## Relationships
- Location ← Products (Product.location_id)
- Location ← VariantInventory (VariantInventory.location_id)
- Location ← StockTransfers (as from_location_id or to_location_id)

---

## Business Logic
- No automatic default-toggling on create/update (is_default is set manually).
- Scoped by `company_id` from JWT context.
- `CreateLocation` uses `ShouldBind` (supports both JSON and form data).
- `UpdateLocation` uses `ShouldBindJSON` (JSON only).

---

## Endpoints

### GET /locations/
**Auth:** Bearer JWT required
**Purpose:** List all locations for the company (no pagination).

**Response 200:**
```json
{
  "message": "Locations retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "name": "Main Warehouse",
      "address": "123 Industrial Ave",
      "contact_person": "Bob",
      "is_default": true,
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

**Error:** `500` — `{"error": "Failed to retrieve locations"}`

---

### GET /locations/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Location fetched successfully",
  "data": { /* location object */ }
}
```

**Error:** `404` — `{"error": "Location not found"}`

---

### POST /locations/
**Auth:** Bearer JWT required
**Content-Type:** `application/json` or `multipart/form-data`

**Request Body:**
```json
{
  "name": "Main Warehouse",
  "address": "123 Industrial Ave",
  "contact_person": "Bob",
  "is_default": true
}
```

**Response 201:**
```json
{
  "message": "Location created successfully",
  "data": { /* location object */ }
}
```

**Error:** `500` — `{"error": "<db error message>"}`

---

### PUT /locations/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Partial update via map of fields.

**Request Body:** Any subset of location fields as JSON object.

**Response 200:**
```json
{
  "message": "Location updated successfully",
  "data": { /* location object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid JSON"}`
- `404` — `{"error": "Location not found"}`
- `500` — `{"error": "Failed to update location"}`

---

### DELETE /locations/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Location deleted successfully"
}
```

**Error Responses:**
- `404` — `{"error": "Location not found"}`
- `500` — `{"error": "Failed to delete location"}`

---

## Dependencies
- **products** — products reference a location via location_id
- **inventory** — variant_inventory records are scoped to a location
- **stock-transfers** — transfers move stock between locations
