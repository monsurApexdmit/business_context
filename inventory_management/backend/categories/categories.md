# Module: Categories

## Business Purpose
Manages the hierarchical product category tree for each company. Categories can have parent-child relationships. Supports tree view (roots with children), flat list, pagination, and status filtering.

---

## Database Table: `categories`

| Column        | Type          | Notes                                             |
|---------------|---------------|---------------------------------------------------|
| id            | uint (PK, AI) |                                                   |
| company_id    | uint (FK)     | FK → companies.id; indexed                        |
| category_name | varchar(100)  | not null                                          |
| parent_id     | *uint (FK)    | FK → categories.id (self-referencing); nullable   |
| status        | bool          | default true (active)                             |
| created_at    | timestamp     | Auto                                              |
| updated_at    | timestamp     | Auto                                              |
| deleted_at    | timestamp     | Soft delete index                                 |

---

## Relationships
- Category → Parent Category (BelongsTo via parent_id, self-reference, optional)
- Category → Children (HasMany via parent_id, self-reference)
- Category ← Products (referenced by products.category_id)

---

## Business Logic
- **Tree view (default):** Returns only root categories (parent_id IS NULL) with their children preloaded
- **Flat view:** Returns all categories (preloads Parent only)
- **All view:** Returns all categories without tree structure
- Prevents deleting a category that has children (must delete/reassign children first)
- Prevents creating a circular reference (category cannot be its own ancestor)
- Duplicate category name within the same company is rejected
- Status toggle: flips current status boolean
- Bulk delete: rejects if any selected category has children

---

## Endpoints

### GET /categories/
**Auth:** Bearer JWT required
**Purpose:** List categories with pagination, search, view mode, and status filtering.

**Query Parameters:**
| Param            | Type   | Default | Description                                    |
|------------------|--------|---------|------------------------------------------------|
| page             | int    | 1       | Pagination page                                |
| limit            | int    | 100     | Items per page (max 100)                       |
| search           | string |         | Filter by category_name (LIKE)                 |
| view             | string | tree    | tree / flat / all                              |
| include_inactive | string | false   | Include inactive categories (true/false)       |

**Response Headers:**
- `X-Total-Count` — total count
- `X-Page-Count` — total pages
- `X-Current-Page` — current page

**Response 200:**
```json
{
  "message": "Categories retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "category_name": "Clothing",
      "parent_id": null,
      "parent": null,
      "children": [
        {
          "id": 2,
          "companyId": 1,
          "category_name": "T-Shirts",
          "parent_id": 1,
          "status": true,
          "created_at": "...",
          "updated_at": "..."
        }
      ],
      "status": true,
      "created_at": "...",
      "updated_at": "..."
    }
  ],
  "pagination": {
    "total": 10,
    "page": 1,
    "limit": 100,
    "total_pages": 1,
    "has_next": false,
    "has_previous": false
  }
}
```

---

### GET /categories/simple
**Auth:** Bearer JWT required
**Purpose:** Returns a minimal flat list (id, category_name, parent_id, status) for dropdown menus.

**Response 200:**
```json
{
  "message": "Categories retrieved successfully",
  "data": [
    { "id": 1, "category_name": "Clothing", "parent_id": null, "status": true }
  ]
}
```

---

### GET /categories/stats
**Auth:** Bearer JWT required
**Purpose:** Returns count statistics.

**Response 200:**
```json
{
  "message": "Category statistics retrieved successfully",
  "data": {
    "total": 15,
    "active": 12,
    "inactive": 3,
    "root_categories": 5,
    "subcategories": 10
  }
}
```

---

### GET /categories/:id
**Auth:** Bearer JWT required
**Path Params:** `id` (uint)

**Response 200:**
```json
{
  "message": "Category fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "category_name": "Clothing",
    "parent_id": null,
    "parent": null,
    "children": [...],
    "status": true,
    "created_at": "...",
    "updated_at": "..."
  }
}
```

**Error Responses:**
- `404` — `{"error": "Category not found"}`

---

### POST /categories/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "category_name": "T-Shirts",
  "parent_id": 1,
  "status": true
}
```

**Validation:**
- `category_name` — required
- `parent_id` — optional uint; must exist in same company if provided
- `status` — optional bool; defaults to true

**Response 201:**
```json
{
  "message": "Category created successfully",
  "data": { /* category object with parent preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid JSON"}` / `{"error": "Parent category not found", "field": "parent_id"}`
- `409` — `{"error": "Category with this name already exists", "field": "category_name"}`
- `422` — validation errors
- `500` — `{"error": "Failed to create category"}`

---

### PUT /categories/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Partial update — only provided fields are changed. `parent_id: null` is explicitly supported to remove parent.

**Request Body (all optional):**
```json
{
  "category_name": "New Name",
  "parent_id": null,
  "status": false
}
```

**Business Rules:**
- Cannot set parent_id to self
- Cannot create circular references (depth limit: 10 levels)
- Duplicate category name rejected

**Response 200:**
```json
{
  "message": "Category updated successfully",
  "data": { /* category object with parent & children preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Category cannot be its own parent", "field": "parent_id"}` / `{"error": "Parent category not found", "field": "parent_id"}` / `{"error": "Cannot create circular category reference", "field": "parent_id"}`
- `404` — `{"error": "Category not found"}`
- `409` — `{"error": "Category with this name already exists", "field": "category_name"}`

---

### PATCH /categories/:id/toggle-status
**Auth:** Bearer JWT required
**Purpose:** Toggles the status boolean (true↔false).

**Response 200:**
```json
{
  "message": "Category status updated successfully",
  "data": { /* category object */ }
}
```

---

### DELETE /categories/:id
**Auth:** Bearer JWT required
**Purpose:** Soft-delete a category. Blocked if it has children.

**Response 200:**
```json
{
  "message": "Category deleted successfully"
}
```

**Error Responses:**
- `400` — `{"error": "Cannot delete category with subcategories. Delete or reassign subcategories first."}`
- `404` — `{"error": "Category not found"}`

---

### POST /categories/bulk-delete
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "ids": [1, 2, 3]
}
```

**Validation:** `ids` — required array, min 1 element

**Response 200:**
```json
{
  "message": "Categories deleted successfully",
  "deleted": 3
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request"}` / `{"error": "Cannot delete categories with subcategories"}`

---

## Dependencies
- **products** — products reference categories via category_id
