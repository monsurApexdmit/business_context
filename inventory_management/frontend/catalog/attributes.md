# Frontend Module: Attributes

## Pages
- `/dashboard/catalog/attributes` — Attribute list with search and management
- `/dashboard/catalog/attributes/new` — Create attribute form
- `/dashboard/catalog/attributes/:id/edit` — Edit attribute form

## API Service
`lib/attributeApi.ts`

---

## GET /attributes/
**Purpose:** Fetch paginated list of product attributes
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by attribute name (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "name": "color",
      "display_name": "Color",
      "option_type": "select | multiselect | text | radio",
      "values": "red,green,blue",
      "status": true,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "deleted_at": null
    }
  ],
  "pagination": {
    "total": 30,
    "page": 1,
    "limit": 20,
    "total_pages": 2,
    "has_next": true,
    "has_previous": false
  }
}
```

**Note:** `values` is a comma-separated string of possible attribute values (e.g., `"red,green,blue"`). Frontend must split on comma to render as chips/tags.

**Frontend Impact:**
- Attribute management table
- `option_type` determines how the attribute input renders in the product form

---

## GET /attributes/simple
**Purpose:** Fetch all active attributes without pagination (for product form dropdowns)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "name": "color",
      "display_name": "Color",
      "option_type": "select",
      "values": "red,green,blue",
      "status": true,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "deleted_at": null
    }
  ]
}
```

**Frontend Impact:**
- Product creation/edit form "Add Attribute" multi-select populated from this endpoint
- User selects attributes by ID; values are shown for each selected attribute

---

## GET /attributes/stats
**Purpose:** Fetch aggregate statistics about attributes
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "...stats fields (shape not strongly typed in frontend — returned as `any`)..."
  }
}
```

**Note:** The TypeScript return type is `{ message: string; data: any }`. Backend shape is not strictly defined in the frontend — document whatever the backend returns.

**Frontend Impact:**
- Stats cards on attribute management page (if implemented)

---

## GET /attributes/:id
**Purpose:** Fetch a single attribute by ID
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric attribute ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "name": "color",
    "display_name": "Color",
    "option_type": "select",
    "values": "red,green,blue",
    "status": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z",
    "deleted_at": null
  }
}
```

**Frontend Impact:**
- Edit form pre-populated
- Values string split by comma for tag input display

---

## POST /attributes/
**Purpose:** Create a new product attribute
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "color",
  "display_name": "Color",
  "option_type": "select",
  "values": "red,green,blue",
  "status": true
}
```

**Note:** `name`, `display_name`, `option_type`, and `values` are required. `values` must be a comma-separated string. `status` defaults to `true` if omitted.

**Expected Response 201:**
```json
{
  "message": "Attribute created successfully",
  "data": {
    "id": 1,
    "name": "color",
    "display_name": "Color",
    "option_type": "select",
    "values": "red,green,blue",
    "status": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z",
    "deleted_at": null
  }
}
```

**Frontend Impact:**
- New attribute immediately available in product form attribute selector
- Attribute list refreshed after creation

---

## PUT /attributes/:id
**Purpose:** Update an existing attribute
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric attribute ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "color",
  "display_name": "Color",
  "option_type": "select",
  "values": "red,green,blue,yellow",
  "status": true
}
```

**Note:** All fields optional for partial update.

**Expected Response 200:**
```json
{
  "message": "Attribute updated successfully",
  "data": { "...AttributeResponse object..." }
}
```

**Frontend Impact:**
- Updated values available immediately in product forms that use this attribute
- Existing products using this attribute may need re-evaluation if values are removed

---

## PATCH /attributes/:id/toggle-status
**Purpose:** Toggle an attribute's active/inactive status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric attribute ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:** None

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...AttributeResponse object with toggled status..." }
}
```

**Frontend Impact:**
- Status toggle in attribute row
- Inactive attributes excluded from product form selectors (depending on `getSimple` endpoint filter)

---

## DELETE /attributes/:id
**Purpose:** Soft-delete an attribute
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric attribute ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Attribute deleted successfully"
}
```

**Frontend Impact:**
- Attribute removed from list
- Products using this attribute retain their data (backend-dependent)

---

## POST /attributes/bulk-delete
**Purpose:** Delete multiple attributes at once
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "ids": [1, 2, 3]
}
```

**Expected Response 200:**
```json
{
  "message": "Attributes deleted successfully"
}
```

**Note:** Response is just `{ message }` — no `data` field, no count returned.

**Frontend Impact:**
- Checkbox multi-select + "Delete Selected" button
- Attribute list refreshed after bulk delete
