# Module Overview

**Module Name:** Categories
**Module Path:** `inventory_management/backend/categories`

This module manages the hierarchical product category tree for each company. Categories support parent-child relationships, tree and flat views, status filtering, pagination, and safe deletion guards.

---

# Business Context

Categories allow merchants to organize their product catalog into a structured hierarchy (e.g., Clothing > T-Shirts > Graphic Tees). Each company maintains its own independent category tree. Categories are referenced by products and drive catalog browsing, filtering, and reporting.

---

# Functional Scope

## Included

- Create, read, update, and delete categories per company
- Self-referencing parent-child hierarchy (unlimited depth, enforced max 10 levels on update)
- Three list view modes: `tree` (default), `flat`, and `all`
- Status toggling (active/inactive)
- Soft delete with child-presence guard
- Bulk delete with child-presence guard
- Paginated list with search and status filtering
- Minimal "simple" list endpoint for UI dropdowns
- Stats endpoint with root category and subcategory counts
- All data scoped by `company_id` extracted from JWT

## Excluded

- Product reassignment when a category is deleted (caller must reassign or delete products first)
- Category icon/image management
- Cross-company category sharing or templates

---

# Architecture Notes

- All category records are scoped to a company via `company_id` injected by JWT middleware.
- The tree view (default) loads only root categories (`parent_id IS NULL`) with their direct children preloaded. It does not recursively fetch deep trees in a single pass beyond one level of children.
- Soft delete uses a `deleted_at` timestamp; soft-deleted records are excluded from all queries by default via ORM scope.
- Circular reference detection enforces a depth limit of 10 levels during updates.
- Duplicate category names within the same company are rejected at the application layer.

---

# Data Model

**Table:** `categories`

| Column          | Type         | Constraints / Notes                                          |
|-----------------|--------------|--------------------------------------------------------------|
| `id`            | uint         | Primary key, auto-increment                                  |
| `company_id`    | uint         | FK → `companies.id`; indexed; scopes all records             |
| `category_name` | varchar(100) | Not null                                                     |
| `parent_id`     | *uint        | FK → `categories.id`; self-referencing; nullable (root categories have no parent) |
| `status`        | bool         | Default `true` (active)                                      |
| `created_at`    | timestamp    | Auto-managed                                                 |
| `updated_at`    | timestamp    | Auto-managed                                                 |
| `deleted_at`    | timestamp    | Nullable; soft delete marker; indexed                        |

**Relationships:**

| Relationship                    | Type       | Notes                                                   |
|---------------------------------|------------|---------------------------------------------------------|
| Category → Parent Category      | BelongsTo  | Self-reference via `parent_id`; optional for root nodes |
| Category → Children             | HasMany    | Self-reference via `parent_id`                          |
| Category ← Products             | Referenced | `products.category_id` references this table            |

---

# API Endpoints

| Method | Path                               | Auth       | Description                                          |
|--------|------------------------------------|------------|------------------------------------------------------|
| GET    | `/categories/`                     | Bearer JWT | Paginated list with search, view mode, status filter |
| GET    | `/categories/simple`               | Bearer JWT | Minimal flat list for UI dropdowns                   |
| GET    | `/categories/stats`                | Bearer JWT | Summary counts (total/active/inactive/root/sub)      |
| GET    | `/categories/:id`                  | Bearer JWT | Single category with parent and children preloaded   |
| POST   | `/categories/`                     | Bearer JWT | Create a new category                                |
| PUT    | `/categories/:id`                  | Bearer JWT | Partial update of a category                         |
| PATCH  | `/categories/:id/toggle-status`    | Bearer JWT | Toggle active/inactive status                        |
| DELETE | `/categories/:id`                  | Bearer JWT | Soft delete; blocked if category has children        |
| POST   | `/categories/bulk-delete`          | Bearer JWT | Soft delete multiple categories; blocked if any has children |

---

# Request Payloads

### GET `/categories/` — Query Parameters

| Parameter          | Type   | Default | Notes                                               |
|--------------------|--------|---------|-----------------------------------------------------|
| `page`             | int    | `1`     |                                                     |
| `limit`            | int    | `100`   | Maximum: `100`                                      |
| `search`           | string | —       | LIKE match on `category_name`                       |
| `view`             | string | `tree`  | `tree` / `flat` / `all`                             |
| `include_inactive` | bool   | `false` | When `true`, includes inactive (status=false) records |

### POST `/categories/` — Create Category

```json
{
  "category_name": "T-Shirts",
  "parent_id": 1,
  "status": true
}
```

| Field           | Type   | Required | Notes                                                         |
|-----------------|--------|----------|---------------------------------------------------------------|
| `category_name` | string | Yes      |                                                               |
| `parent_id`     | uint   | No       | Must reference an existing category in the same company       |
| `status`        | bool   | No       | Defaults to `true`                                            |

### PUT `/categories/:id` — Update Category

All fields optional. Passing `"parent_id": null` explicitly removes the parent (promotes to root).

```json
{
  "category_name": "New Name",
  "parent_id": null,
  "status": false
}
```

### POST `/categories/bulk-delete`

```json
{
  "ids": [1, 2, 3]
}
```

`ids` is required; minimum 1 element.

---

# Response Contracts

### GET `/categories/` — `200 OK`

**Response Headers:**

| Header           | Description             |
|------------------|-------------------------|
| `X-Total-Count`  | Total matching records  |
| `X-Page-Count`   | Total number of pages   |
| `X-Current-Page` | Current page number     |

**Response Body (tree view example):**

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
          "created_at": "2024-01-01T00:00:00Z",
          "updated_at": "2024-01-01T00:00:00Z"
        }
      ],
      "status": true,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
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

### GET `/categories/simple` — `200 OK`

```json
{
  "message": "Categories retrieved successfully",
  "data": [
    { "id": 1, "category_name": "Clothing", "parent_id": null, "status": true }
  ]
}
```

### GET `/categories/stats` — `200 OK`

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

### GET `/categories/:id` — `200 OK`

```json
{
  "message": "Category fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "category_name": "Clothing",
    "parent_id": null,
    "parent": null,
    "children": [],
    "status": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

### POST `/categories/` — `201 Created`

```json
{
  "message": "Category created successfully",
  "data": { }
}
```

Returns the created category object with `parent` preloaded.

### PUT `/categories/:id` — `200 OK`

```json
{
  "message": "Category updated successfully",
  "data": { }
}
```

Returns the updated category with `parent` and `children` preloaded.

### PATCH `/categories/:id/toggle-status` — `200 OK`

```json
{
  "message": "Category status updated successfully",
  "data": { }
}
```

### DELETE `/categories/:id` — `200 OK`

```json
{
  "message": "Category deleted successfully"
}
```

### POST `/categories/bulk-delete` — `200 OK`

```json
{
  "message": "Categories deleted successfully",
  "deleted": 3
}
```

---

# Validation Rules

| Rule                                               | HTTP Response      |
|----------------------------------------------------|--------------------|
| `category_name` is required on create             | `422`              |
| `parent_id` must exist in the same company        | `400`              |
| `category_name` must be unique per company        | `409`              |
| Category cannot be its own parent                 | `400`              |
| No circular ancestor references (max depth: 10)   | `400`              |
| Cannot delete a category that has children        | `400`              |
| Bulk delete rejected if any category has children | `400`              |
| `ids` array must have at least 1 element          | `400`              |
| `limit` capped at `100`                           | Enforced server-side |

---

# Business Logic Details

- `company_id` is always injected from JWT middleware; it is never accepted from the client.
- **Tree view (default):** Returns root categories (`parent_id IS NULL`) only, with their direct children preloaded. Deep nesting beyond one child level is not expanded in this response.
- **Flat view:** Returns all categories with `Parent` preloaded.
- **All view:** Returns all categories without any tree structure or parent preloading.
- **Status toggle:** Flips the current `status` boolean; no request body is required.
- **Circular reference check:** Before allowing a `parent_id` update, the system walks up the ancestor chain up to 10 levels deep to ensure no cycle would be created.
- **Delete guard:** A category with at least one child category cannot be deleted. The caller must delete or reassign child categories first.
- **Bulk delete guard:** All supplied IDs are checked for children before any deletion is performed; the entire operation is rejected if any ID has children.
- **Soft delete:** Sets `deleted_at`; soft-deleted records are excluded from all queries automatically.

---

# Dependencies

| Module / Table | Direction    | Notes                                         |
|----------------|--------------|-----------------------------------------------|
| `companies`    | FK (inbound) | `company_id` scopes all category records      |
| `products`     | Referenced   | `products.category_id` references this table  |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is extracted from the JWT and used to scope all database queries; users cannot access or modify categories belonging to other companies.
- No role-based access control is documented at the category level (see Open Questions).

---

# Error Handling

| HTTP Status | Scenario                                                                     |
|-------------|------------------------------------------------------------------------------|
| `400`       | Invalid JSON; parent category not found; self-reference; circular reference; delete blocked by children |
| `401`       | Missing or invalid JWT token                                                 |
| `404`       | Category not found for the given `:id`                                       |
| `409`       | Duplicate `category_name` within the same company                            |
| `422`       | Validation errors (missing required fields)                                  |
| `500`       | Unexpected server or database error                                          |

**Error response examples:**

```json
{ "error": "Category not found" }
{ "error": "Category with this name already exists", "field": "category_name" }
{ "error": "Category cannot be its own parent", "field": "parent_id" }
{ "error": "Cannot create circular category reference", "field": "parent_id" }
{ "error": "Cannot delete category with subcategories. Delete or reassign subcategories first." }
{ "error": "Parent category not found", "field": "parent_id" }
```

---

# Testing Notes

- Test tree view: only root categories should be returned, each with their direct children.
- Test flat view: all categories returned with `parent` preloaded.
- Test `all` view: all categories without parent or children preloading.
- Test `include_inactive`: inactive categories must be excluded by default.
- Test duplicate name within the same company returns `409`; same name in a different company must succeed.
- Test self-referential parent assignment returns `400`.
- Test circular reference at depth 2, 5, and 10; test that depth > 10 is also rejected.
- Test delete blocked when category has children.
- Test bulk delete blocked when any of the IDs has children.
- Test setting `parent_id: null` on update promotes the category to root.
- Test stats counts match actual active/inactive/root/subcategory records after mutations.

---

# Open Questions / Missing Details

- **Tree view depth:** The tree view preloads direct children, but it is unclear whether grandchildren and deeper levels are included. The behavior for deeply nested categories in tree view should be confirmed.
- **Role-based access control:** No documentation on whether specific SaaS roles have differing permissions on category endpoints.
- **Product behavior on category delete:** There is no documented behavior for what happens to products assigned to a soft-deleted category. It is unclear whether they become uncategorized or are blocked.
- **Bulk delete atomicity:** It is unclear whether the bulk delete check-and-delete is performed in a single database transaction.

---

# Change Log / Notes

- Initial documentation authored from source specification.
