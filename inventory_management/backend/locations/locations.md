# Module Overview

**Module Name:** Locations
**Module Path:** `inventory_management/backend/locations`

This module manages warehouse and store locations for a company. Locations are used to scope inventory quantities, product assignments, and stock transfers. One location per company can be designated as the default.

---

# Business Context

A company may operate across multiple warehouses or store locations. The Locations module provides the reference data that ties inventory, products, and stock movements to a physical place. Every variant's stock quantity is tracked per location via the `variant_inventory` table, and stock transfers move inventory between locations.

---

# Functional Scope

## Included

- Create, read, update, and delete location records per company
- Support for a default location flag (`is_default`)
- Soft delete
- All data scoped by `company_id` extracted from JWT

## Excluded

- Automatic reassignment of the default location when a default is deleted or changed (callers manage `is_default` manually)
- Pagination (the list endpoint returns all locations for the company)
- Inventory quantity management (handled by the Inventory and Stock Transfers modules)
- Multi-location stock reporting (handled by the Inventory module)

---

# Architecture Notes

- All location records are scoped to a company via `company_id` injected by JWT middleware.
- `is_default` is set and managed manually by the caller; there is no automatic toggling or enforcement of a single default per company.
- The create endpoint (`POST /locations/`) uses `ShouldBind`, which accepts both JSON and `multipart/form-data`.
- The update endpoint (`PUT /locations/:id`) uses `ShouldBindJSON` and accepts JSON only.
- Soft delete is implemented via a `deleted_at` timestamp.

---

# Data Model

**Table:** `locations`

| Column           | Type      | Constraints / Notes                                     |
|------------------|-----------|---------------------------------------------------------|
| `id`             | uint      | Primary key, auto-increment; JSON key: `id`             |
| `company_id`     | uint      | FK → `companies.id`; indexed; JSON key: `companyId`     |
| `name`           | string    | JSON key: `name`                                        |
| `address`        | string    | JSON key: `address`                                     |
| `contact_person` | string    | JSON key: `contact_person`                              |
| `is_default`     | bool      | Default `false`; JSON key: `is_default`                 |
| `created_at`     | timestamp | JSON key: `created_at`                                  |
| `updated_at`     | timestamp | JSON key: `updated_at`                                  |
| `deleted_at`     | timestamp | Soft delete; JSON key: `deleted_at`                     |

**Relationships:**

| Relationship                        | Type       | Notes                                                         |
|-------------------------------------|------------|---------------------------------------------------------------|
| Location ← Products                 | Referenced | `products.location_id` references this table                  |
| Location ← VariantInventory         | Referenced | `variant_inventory.location_id`; per-location stock quantities |
| Location ← StockTransfers (source)  | Referenced | `stock_transfers.from_location_id`                            |
| Location ← StockTransfers (dest)    | Referenced | `stock_transfers.to_location_id`                              |

---

# API Endpoints

| Method | Path              | Auth       | Description                                         |
|--------|-------------------|------------|-----------------------------------------------------|
| GET    | `/locations/`     | Bearer JWT | List all locations for the company (no pagination)  |
| GET    | `/locations/:id`  | Bearer JWT | Single location by ID                               |
| POST   | `/locations/`     | Bearer JWT | Create a new location                               |
| PUT    | `/locations/:id`  | Bearer JWT | Partial update of a location (JSON only)            |
| DELETE | `/locations/:id`  | Bearer JWT | Soft delete a location                              |

---

# Request Payloads

### POST `/locations/` — Create Location

Accepts `application/json` or `multipart/form-data`.

```json
{
  "name": "Main Warehouse",
  "address": "123 Industrial Ave",
  "contact_person": "Bob",
  "is_default": true
}
```

| Field            | Type   | Required | Notes                                                        |
|------------------|--------|----------|--------------------------------------------------------------|
| `name`           | string | No       | Not explicitly marked required; see Open Questions           |
| `address`        | string | No       |                                                              |
| `contact_person` | string | No       |                                                              |
| `is_default`     | bool   | No       | Defaults to `false`; set manually; no auto-toggle enforced   |

### PUT `/locations/:id` — Update Location

Accepts `application/json` only. Provide any subset of location fields. Fields not supplied are left unchanged (partial update via map).

---

# Response Contracts

### GET `/locations/` — `200 OK`

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
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

No pagination; all locations for the company are returned in a single response.

### GET `/locations/:id` — `200 OK`

```json
{
  "message": "Location fetched successfully",
  "data": { }
}
```

### POST `/locations/` — `201 Created`

```json
{
  "message": "Location created successfully",
  "data": { }
}
```

### PUT `/locations/:id` — `200 OK`

```json
{
  "message": "Location updated successfully",
  "data": { }
}
```

### DELETE `/locations/:id` — `200 OK`

```json
{
  "message": "Location deleted successfully"
}
```

---

# Validation Rules

| Rule                                            | HTTP Response |
|-------------------------------------------------|---------------|
| `:id` must correspond to an existing location   | `404`         |
| Request body must be valid JSON (PUT)           | `400`         |

> No documented server-side validation for required fields on create. See Open Questions.

---

# Business Logic Details

- `company_id` is always injected from JWT middleware; it is never accepted from the client.
- `is_default` is a simple boolean flag. There is no enforcement that only one location per company holds `is_default = true`; callers are responsible for managing this.
- The create endpoint accepts both JSON and form data (`ShouldBind`); the update endpoint accepts JSON only (`ShouldBindJSON`).
- Soft delete sets `deleted_at`; soft-deleted records are excluded from all queries automatically.
- No cascade behavior is documented for deletion: if a location is deleted, existing `product.location_id`, `variant_inventory.location_id`, or `stock_transfers` references are not automatically cleared or reassigned.

---

# Dependencies

| Module / Table     | Direction    | Notes                                                              |
|--------------------|--------------|---------------------------------------------------------------------|
| `companies`        | FK (inbound) | `company_id` scopes all location records                           |
| `products`         | Referenced   | `products.location_id` references this table                       |
| `variant_inventory`| Referenced   | Per-location stock quantities reference `location_id`              |
| `stock_transfers`  | Referenced   | Transfers reference `from_location_id` and `to_location_id`        |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is extracted from the JWT and used to scope all queries; users cannot access or modify locations belonging to other companies.
- No role-based access control is documented at the location module level (see Open Questions).

---

# Error Handling

| HTTP Status | Scenario                                           |
|-------------|----------------------------------------------------|
| `400`       | Invalid JSON body on PUT                           |
| `401`       | Missing or invalid JWT token                       |
| `404`       | Location not found for the given `:id`             |
| `500`       | Unexpected server or database error                |

**Error response examples:**

```json
{ "error": "Location not found" }
{ "error": "Invalid JSON" }
{ "error": "Failed to update location" }
{ "error": "Failed to delete location" }
{ "error": "Failed to retrieve locations" }
```

> Note: The `POST /locations/` 500 response returns the raw database error message rather than a sanitized message. This may expose internal details and should be reviewed.

---

# Testing Notes

- Test that listing locations returns all locations for the authenticated company only.
- Test create with JSON body; test create with `multipart/form-data` body.
- Test update with JSON body; confirm `multipart/form-data` is rejected by PUT.
- Test that `is_default` can be set to `true` on multiple locations simultaneously (no uniqueness constraint).
- Test soft delete; confirm deleted location does not appear in the list.
- Test that attempting to GET, PUT, or DELETE a non-existent location returns `404`.
- Test that `company_id` scoping prevents cross-company access.

---

# Open Questions / Missing Details

- **Required fields on create:** No fields are documented as required for `POST /locations/`. Confirm whether `name` is required or if a location can be created with an empty name.
- **`is_default` uniqueness:** There is no uniqueness constraint or auto-toggle for `is_default`. Confirm whether the application should enforce a single default location per company.
- **Cascade on delete:** No documented behavior for what happens to `products`, `variant_inventory`, and `stock_transfers` records that reference a deleted location. Confirm whether orphan records are handled.
- **Raw error exposure on create:** The 500 response for `POST /locations/` returns `{"error": "<db error message>"}` — the raw database error string. This should be sanitized in production.
- **Role-based access control:** No documentation on whether specific SaaS roles have differing permissions on location endpoints.
- **No `name` uniqueness constraint:** It is not documented whether two locations in the same company can share the same name.

---

# Change Log / Notes

- Initial documentation authored from source specification.
- Raw database error on create 500 response is a known issue; flagged for review.
