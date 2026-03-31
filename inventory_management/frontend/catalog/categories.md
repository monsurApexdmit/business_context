# Frontend Module: Categories

## Pages
- `/dashboard/catalog/categories` — Category tree/list management
- `/dashboard/catalog/categories/new` — Create category form
- `/dashboard/catalog/categories/:id/edit` — Edit category form

## API Service
`lib/categoryApi.ts`

---

## GET /categories/
**Purpose:** Fetch all categories with optional tree/flat view and filters
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by category name (optional)
- `view` — `"tree"` | `"flat"` | `"all"` (optional, controls response structure)
- `include_inactive` — boolean, include inactive categories (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "category_name": "Electronics",
      "parent_id": null,
      "parent": null,
      "children": [
        {
          "id": 2,
          "category_name": "Phones",
          "parent_id": 1,
          "status": true,
          "created_at": "2024-01-01T00:00:00Z",
          "updated_at": "2024-01-01T00:00:00Z",
          "deleted_at": null
        }
      ],
      "status": true,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "deleted_at": null
    }
  ],
  "pagination": {
    "total": 50,
    "page": 1,
    "limit": 20,
    "total_pages": 3,
    "has_next": true,
    "has_previous": false
  }
}
```

**Note:** `parent` and `children` fields appear depending on the `view` param. `deleted_at` is present for soft-deleted records.

**Frontend Impact:**
- Category tree component renders from `children` nesting
- Flat list view renders all in a table
- Status badge shows active/inactive

---

## GET /categories/simple
**Purpose:** Fetch minimal category list for dropdown selectors (no pagination overhead)
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
      "category_name": "Electronics",
      "parent_id": null,
      "status": true,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

**Frontend Impact:**
- Product creation form "Category" dropdown populated from this endpoint
- Lightweight — only called when a dropdown is rendered, not for the full category management page

---

## GET /categories/stats
**Purpose:** Fetch aggregate statistics about categories
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "total": 50,
    "active": 42,
    "inactive": 8,
    "root_categories": 10,
    "subcategories": 40
  }
}
```

**Frontend Impact:**
- Stats cards at top of categories page
- `root_categories` vs `subcategories` breakdown shown in summary

---

## GET /categories/:id
**Purpose:** Fetch a single category by ID
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric category ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "category_name": "Electronics",
    "parent_id": null,
    "parent": null,
    "children": [],
    "status": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z",
    "deleted_at": null
  }
}
```

**Frontend Impact:**
- Edit form pre-populated
- Parent category selector shows current parent

---

## POST /categories/
**Purpose:** Create a new category
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "category_name": "Electronics",
  "parent_id": null,
  "status": true
}
```

**Note:** `parent_id` is always explicitly sent — even if null — to distinguish from "not provided". Frontend normalizes `undefined` to `null` before sending.

**Expected Response 201:**
```json
{
  "message": "Category created successfully",
  "data": {
    "id": 1,
    "category_name": "Electronics",
    "parent_id": null,
    "status": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z",
    "deleted_at": null
  }
}
```

**Frontend Impact:**
- New category appears in category tree
- Success toast shown
- If `parent_id` is set, new category appears as child of that parent

---

## PUT /categories/:id
**Purpose:** Update an existing category
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric category ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "category_name": "Updated Name",
  "parent_id": 2,
  "status": true
}
```

**Note:** All fields optional. `parent_id` is only included in payload if it was provided in the update data (including explicit `null` to remove parent). Frontend normalizes `undefined` to `null` if key is present.

**Expected Response 200:**
```json
{
  "message": "Category updated successfully",
  "data": { "...CategoryResponse object..." }
}
```

**Frontend Impact:**
- Category tree re-renders with updated name/parent
- Changing `parent_id` moves the category in the tree

---

## PATCH /categories/:id/toggle-status
**Purpose:** Toggle a category's active/inactive status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric category ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:** None

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...CategoryResponse object with toggled status..." }
}
```

**Frontend Impact:**
- Status toggle in category row flips without form submission
- Inactive categories may be hidden from product creation dropdowns (depends on `include_inactive` param)

---

## DELETE /categories/:id
**Purpose:** Soft-delete a category
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric category ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Category deleted successfully"
}
```

**Frontend Impact:**
- Category removed from tree/list
- Products in this category may become uncategorized (backend-dependent)

---

## POST /categories/bulk-delete
**Purpose:** Delete multiple categories at once
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
  "message": "Categories deleted successfully",
  "deleted": 3
}
```

**Note:** Response shape `{ message, deleted }` — NOT the standard `{ message, data }` envelope.

**Frontend Impact:**
- Checkbox-select-all + "Delete Selected" button pattern
- `deleted` count shown in success toast ("3 categories deleted")
- Category list refreshed after bulk delete
