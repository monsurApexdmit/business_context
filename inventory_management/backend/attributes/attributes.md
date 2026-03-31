# Module: Attributes

## Business Purpose
Manages product attribute definitions (e.g., Size, Color, Material). Attributes define the axes for product variants. Each attribute has an option type (text, dropdown, radio, checkbox, color, size) and an optional list of predefined values.

---

## Database Table: `attributes`

| Column       | Type          | Notes                                                                                 |
|--------------|---------------|---------------------------------------------------------------------------------------|
| id           | uint (PK, AI) |                                                                                       |
| company_id   | uint (FK)     | FK → companies.id; indexed                                                            |
| name         | varchar(100)  | not null; unique per company: idx_attribute_company_name (composite with company_id) |
| display_name | varchar(150)  | not null                                                                              |
| option_type  | varchar(50)   | default 'text'; values: text, dropdown, radio, checkbox, color, size                 |
| values       | text          | JSON array of possible values for dropdown/radio/checkbox                             |
| description  | text          |                                                                                       |
| is_required  | bool          | default false                                                                         |
| status       | bool          | default true                                                                          |
| sort_order   | int           | default 0                                                                             |
| created_at   | timestamp     | Auto                                                                                  |
| updated_at   | timestamp     | Auto                                                                                  |
| deleted_at   | timestamp     | Soft delete                                                                           |

---

## Relationships
- Attributes ↔ Products (ManyToMany via `product_attributes` join table)

---

## Endpoints

### GET /attributes/
**Auth:** Bearer JWT required
**Purpose:** List attributes with pagination, search, and status filtering.

**Query Parameters:**
| Param            | Type   | Default | Description                            |
|------------------|--------|---------|----------------------------------------|
| page             | int    | 1       |                                        |
| limit            | int    | 100     | max 100                                |
| search           | string |         | Filter by name or display_name (LIKE)  |
| view             | string | tree    | (reserved; behaves same as flat)       |
| include_inactive | string | false   | Include inactive attributes            |

**Response Headers:** `X-Total-Count`, `X-Page-Count`, `X-Current-Page`

**Response 200:**
```json
{
  "message": "Attributes retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "name": "size",
      "display_name": "Size",
      "option_type": "dropdown",
      "values": "[\"XS\",\"S\",\"M\",\"L\",\"XL\"]",
      "description": "Clothing size",
      "is_required": true,
      "status": true,
      "sort_order": 0,
      "created_at": "...",
      "updated_at": "..."
    }
  ],
  "pagination": {
    "total": 5,
    "page": 1,
    "limit": 100,
    "total_pages": 1,
    "has_next": false,
    "has_previous": false
  }
}
```

---

### GET /attributes/simple
**Auth:** Bearer JWT required
**Purpose:** Minimal list of active attributes for use in dropdowns.

**Response 200:**
```json
{
  "message": "Attributes retrieved successfully",
  "data": [
    { "id": 1, "name": "size", "display_name": "Size", "option_type": "dropdown", "values": "...", "status": true }
  ]
}
```

---

### GET /attributes/stats
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Attribute statistics retrieved successfully",
  "data": {
    "total": 8,
    "active": 6,
    "inactive": 2,
    "root_categories": 0,
    "subcategories": 0
  }
}
```

> Note: `root_categories` and `subcategories` fields are 0 (not applicable to attributes).

---

### GET /attributes/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Attribute fetched successfully",
  "data": { /* attribute object */ }
}
```

**Error:** `404` — `{"error": "Attribute not found"}`

---

### POST /attributes/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "name": "color",
  "display_name": "Color",
  "option_type": "color",
  "values": "[\"Red\",\"Blue\",\"Green\"]",
  "description": "Product color",
  "is_required": false,
  "status": true,
  "sort_order": 1
}
```

**Validation:**
- `name` — required
- `display_name` — required
- `option_type` — required; one of: text, dropdown, radio, checkbox, color, size
- `values` — optional string (JSON array)

**Response 201:**
```json
{
  "message": "Attribute created successfully",
  "data": { /* attribute object */ }
}
```

**Error Responses:**
- `409` — `{"error": "Attribute with this name already exists", "field": "name"}`
- `422` — validation errors: `{"message": "Validation failed", "errors": [{"field": "name", "message": "..."}]}`

---

### PUT /attributes/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Full update (all fields optional in the request).

**Request Body:** Same fields as POST (all optional).

**Response 200:**
```json
{
  "message": "Attribute updated successfully",
  "data": { /* attribute object */ }
}
```

---

### PATCH /attributes/:id/toggle-status
**Auth:** Bearer JWT required
**Purpose:** Toggles status boolean.

**Response 200:**
```json
{
  "message": "Attribute status updated successfully",
  "data": { /* attribute object */ }
}
```

---

### DELETE /attributes/:id
**Auth:** Bearer JWT required
**Purpose:** Soft-delete an attribute.

**Response 200:**
```json
{
  "message": "Attribute deleted successfully"
}
```

---

### POST /attributes/bulk-delete
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "ids": [1, 2, 3]
}
```

**Response 200:**
```json
{
  "message": "Attributes deleted successfully",
  "deleted": 3
}
```

---

## Dependencies
- **products** — attributes are linked to products via product_attributes join table
