# Module Overview

**Module Name:** Attributes
**Module Path:** `inventory_management/backend/attributes`

This module manages product attribute definitions (e.g., Size, Color, Material). Attributes define the axes along which product variants differ. Each attribute specifies an option type and an optional set of predefined values.

---

# Business Context

Attributes are the building blocks for product variant configuration. When a product has multiple variants (e.g., a shirt available in different sizes and colors), each axis of variation is represented by an attribute. Merchants define attributes at the company level; these are then linked to products to drive variant generation and UI rendering.

---

# Functional Scope

## Included

- Create, read, update, and delete attribute definitions per company
- Support for multiple option types: `text`, `dropdown`, `radio`, `checkbox`, `color`, `size`
- Optional list of predefined values for applicable option types (dropdown, radio, checkbox)
- Soft delete with status toggling
- Bulk delete operation
- Paginated list with search and status filtering
- Minimal "simple" list endpoint for UI dropdowns
- Stats endpoint for dashboard summary counts
- All data scoped by `company_id` extracted from JWT

## Excluded

- Attribute value management as independent records (values are stored as a JSON array on the attribute row itself)
- Tree view (the `view=tree` query parameter is reserved but not implemented)
- Variant generation logic (handled by the Products module)
- Attribute inheritance or sharing across companies

---

# Architecture Notes

- All attribute records are scoped to a company via `company_id` injected by JWT middleware; clients cannot supply or spoof this value.
- The `values` column stores a JSON-encoded array of strings, used for option types that present a predefined list (dropdown, radio, checkbox).
- `option_type` determines how the attribute is rendered in the UI and which predefined values are applicable.
- Soft delete is implemented via a `deleted_at` timestamp; deleted records are excluded from all queries by default via ORM scope.
- The `view=tree` query parameter on the list endpoint is reserved for future use and currently has no effect.
- The stats endpoint always returns `root_categories: 0` and `subcategories: 0`; these fields are structural artifacts inherited from a shared stats response struct and are not meaningful for attributes.

---

# Data Model

**Table:** `attributes`

| Column         | Type         | Constraints / Notes                                                                      |
|----------------|--------------|------------------------------------------------------------------------------------------|
| `id`           | uint         | Primary key, auto-increment                                                              |
| `company_id`   | uint         | FK → `companies.id`; indexed; scopes all records to a company                           |
| `name`         | varchar(100) | Not null; composite unique index with `company_id` (`idx_attribute_company_name`)        |
| `display_name` | varchar(150) | Not null; human-readable label shown in UI                                               |
| `option_type`  | varchar(50)  | Default `'text'`; allowed values: `text`, `dropdown`, `radio`, `checkbox`, `color`, `size` |
| `values`       | text         | JSON-encoded array of predefined option values; applicable to dropdown/radio/checkbox    |
| `description`  | text         | Optional free-text description                                                           |
| `is_required`  | bool         | Default `false`; whether this attribute must be assigned when creating a variant         |
| `status`       | bool         | Default `true` (active)                                                                  |
| `sort_order`   | int          | Default `0`; controls display ordering                                                   |
| `created_at`   | timestamp    | Auto-managed                                                                             |
| `updated_at`   | timestamp    | Auto-managed                                                                             |
| `deleted_at`   | timestamp    | Nullable; soft delete marker                                                             |

**Relationships:**

| Relationship            | Type         | Details                                            |
|-------------------------|--------------|----------------------------------------------------|
| Attributes ↔ Products   | Many-to-Many | Joined via `product_attributes` join table         |

---

# API Endpoints

| Method | Path                              | Auth       | Description                                   |
|--------|-----------------------------------|------------|-----------------------------------------------|
| GET    | `/attributes/`                    | Bearer JWT | Paginated list with search and status filter  |
| GET    | `/attributes/simple`              | Bearer JWT | Minimal active list for UI dropdowns          |
| GET    | `/attributes/stats`               | Bearer JWT | Summary counts (total/active/inactive)        |
| GET    | `/attributes/:id`                 | Bearer JWT | Single attribute by ID                        |
| POST   | `/attributes/`                    | Bearer JWT | Create a new attribute                        |
| PUT    | `/attributes/:id`                 | Bearer JWT | Update an attribute                           |
| PATCH  | `/attributes/:id/toggle-status`   | Bearer JWT | Toggle active/inactive status                 |
| DELETE | `/attributes/:id`                 | Bearer JWT | Soft delete a single attribute                |
| POST   | `/attributes/bulk-delete`         | Bearer JWT | Soft delete multiple attributes by ID array   |

---

# Request Payloads

### GET `/attributes/` — Query Parameters

| Parameter          | Type   | Default | Notes                                              |
|--------------------|--------|---------|----------------------------------------------------|
| `page`             | int    | `1`     |                                                    |
| `limit`            | int    | `100`   | Maximum: `100`                                     |
| `search`           | string | —       | LIKE match on `name` or `display_name`             |
| `view`             | string | —       | Reserved; `tree` value is accepted but has no effect |
| `include_inactive` | bool   | —       | When set, includes records where `status = false`  |

### POST `/attributes/` — Create Attribute

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

| Field          | Type   | Required | Notes                                                                  |
|----------------|--------|----------|------------------------------------------------------------------------|
| `name`         | string | Yes      | Must be unique per company                                             |
| `display_name` | string | Yes      |                                                                        |
| `option_type`  | string | Yes      | One of: `text`, `dropdown`, `radio`, `checkbox`, `color`, `size`      |
| `values`       | string | No       | JSON-encoded array string; relevant for dropdown/radio/checkbox types  |
| `description`  | string | No       |                                                                        |
| `is_required`  | bool   | No       | Defaults to `false`                                                    |
| `status`       | bool   | No       | Defaults to `true`                                                     |
| `sort_order`   | int    | No       | Defaults to `0`                                                        |

### PUT `/attributes/:id` — Update Attribute

Same fields as POST; all fields are optional in the request body.

### POST `/attributes/bulk-delete`

```json
{
  "ids": [1, 2, 3]
}
```

---

# Response Contracts

### GET `/attributes/` — Paginated List

**Response Headers:**

| Header           | Description             |
|------------------|-------------------------|
| `X-Total-Count`  | Total matching records  |
| `X-Page-Count`   | Total number of pages   |
| `X-Current-Page` | Current page number     |

**Response Body (`200 OK`):**

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
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
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

### GET `/attributes/simple` — `200 OK`

```json
{
  "message": "Attributes retrieved successfully",
  "data": [
    {
      "id": 1,
      "name": "size",
      "display_name": "Size",
      "option_type": "dropdown",
      "values": "[\"XS\",\"S\",\"M\",\"L\",\"XL\"]",
      "status": true
    }
  ]
}
```

### GET `/attributes/stats` — `200 OK`

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

> `root_categories` and `subcategories` are always `0`; these fields are not applicable to attributes.

### GET `/attributes/:id` — `200 OK`

```json
{
  "message": "Attribute fetched successfully",
  "data": { }
}
```

### POST `/attributes/` — `201 Created`

```json
{
  "message": "Attribute created successfully",
  "data": { }
}
```

### PUT `/attributes/:id` — `200 OK`

```json
{
  "message": "Attribute updated successfully",
  "data": { }
}
```

### PATCH `/attributes/:id/toggle-status` — `200 OK`

```json
{
  "message": "Attribute status updated successfully",
  "data": { }
}
```

### DELETE `/attributes/:id` — `200 OK`

```json
{
  "message": "Attribute deleted successfully"
}
```

### POST `/attributes/bulk-delete` — `200 OK`

```json
{
  "message": "Attributes deleted successfully",
  "deleted": 3
}
```

---

# Validation Rules

| Rule                                          | HTTP Response                  |
|-----------------------------------------------|--------------------------------|
| `name` is required                            | `422 Unprocessable Entity`     |
| `display_name` is required                    | `422 Unprocessable Entity`     |
| `option_type` is required                     | `422 Unprocessable Entity`     |
| `option_type` must be one of the allowed set  | `422 Unprocessable Entity`     |
| `name` must be unique per `company_id`        | `409 Conflict`                 |
| `limit` query param capped at `100`           | Enforced server-side           |

---

# Business Logic Details

- `company_id` is always injected from JWT middleware; it is never accepted from the client request body.
- The status toggle (`PATCH /toggle-status`) flips the current boolean value; no request body is needed.
- The `values` field is stored as a raw JSON string; the application layer is responsible for serializing and deserializing it.
- Soft delete sets `deleted_at`; soft-deleted records are excluded from all standard queries via ORM scope.
- Bulk delete applies soft delete to all matched IDs belonging to the authenticated company.
- Stats counts (`total`, `active`, `inactive`) reflect only records belonging to the current company.

---

# Dependencies

| Module / Table       | Direction    | Notes                                                       |
|----------------------|--------------|-------------------------------------------------------------|
| `companies`          | FK (inbound) | `company_id` scopes all attribute records                   |
| `products`           | Many-to-Many | Linked via `product_attributes` join table                  |
| `product_attributes` | Join table   | Associates attributes with products for variant configuration |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is extracted from the JWT and used to scope all database queries; users cannot access or modify attributes belonging to other companies.
- No role-based access control is documented at the attribute level (see Open Questions).

---

# Error Handling

| HTTP Status | Scenario                                                             |
|-------------|----------------------------------------------------------------------|
| `400`       | Invalid request body or malformed JSON                               |
| `401`       | Missing or invalid JWT token                                         |
| `404`       | Attribute not found for the given `:id`                              |
| `409`       | Duplicate `name` within the same company                             |
| `422`       | Validation errors (missing required fields, invalid `option_type`)   |
| `500`       | Unexpected server or database error                                  |

**Error response format:**

```json
{
  "error": "Attribute with this name already exists",
  "field": "name"
}
```

Validation errors:

```json
{
  "message": "Validation failed",
  "errors": [
    { "field": "name", "message": "name is required" }
  ]
}
```

---

# Testing Notes

- Test name uniqueness enforcement: the same name must be rejected within a company but accepted across different companies.
- Test all six `option_type` values; verify that unsupported values return `422`.
- Test the `values` field with and without content for each option type.
- Test `include_inactive` filter — confirm inactive records are excluded by default and included when the flag is set.
- Test `view=tree` to confirm the endpoint does not error (parameter is reserved and inert).
- Test bulk delete with a mix of valid and invalid IDs.
- Test the stats endpoint for correct counts after creates, soft deletes, and status toggles.
- Verify soft-deleted records do not appear in list responses or GET by ID.
- Confirm bulk delete only affects records belonging to the authenticated company.

---

# Open Questions / Missing Details

- **PUT semantics:** Documented as "full update (all fields optional)." It is unclear whether omitted fields are set to zero values/defaults or left unchanged. Partial patch vs. full replace behavior should be confirmed with the implementation.
- **`view=tree` future design:** The tree view is reserved. Expected hierarchy structure and response format are not yet defined.
- **Role-based access control:** No documentation exists on whether specific SaaS roles (owner/admin/manager/staff) have differing permissions on attribute endpoints.
- **`values` response format:** The `values` field is stored and returned as a raw JSON string. It is unclear whether the API ever deserializes it into a parsed array in any response.
- **Bulk delete cross-company safety:** Confirm that the bulk delete handler validates all submitted IDs belong to the authenticated company before proceeding.

---

# Change Log / Notes

- Initial documentation authored from source specification.
- `root_categories` and `subcategories` in the stats response are always `0`; likely inherited from a shared response struct. Flagged for review and potential removal.
