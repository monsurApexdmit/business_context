# Module Overview

**Module Name:** Vendors
**Scope:** Backend — Inventory Management
**Purpose:** Manages vendor (supplier) records for a company. Each vendor is paired with a linked User account (role_id = 4) to support potential vendor portal access. Provides full CRUD operations scoped to the authenticated company.

---

# Business Context

Vendors represent external suppliers from whom the company sources products. Each vendor record is tied to a specific company and is isolated by `company_id` derived from the JWT. Beyond basic contact information, each vendor has a corresponding User account automatically created at the time of vendor creation, allowing vendors to potentially log into a portal. Financial fields (`totalPaid`, `amountPayable`) track payment obligations to the vendor over time.

---

# Functional Scope

## Included

- Create, read, update, and soft-delete vendor records scoped to the authenticated company
- Automatic creation of a linked `users` record (role_id = 4) on vendor creation with a default password
- Synchronization of vendor `name` and `email` to the linked user record on update
- Soft-deletion of the linked user when a vendor is deleted
- Filtering vendors by status (`Active`, `Inactive`, `Blocked`, or `all`)
- Paginated and searchable vendor listing (search on `name`, `email`, `phone`)
- Preloading of linked User and Role data when retrieving a single vendor

## Excluded

- Vendor portal authentication flows (handled by the auth module)
- Payment processing or invoice management (financial totals are stored but not computed here)
- Product management (products reference vendors via `vendor_id` but are managed by the products module)
- Vendor return management (handled by the vendor-returns module)

---

# Architecture Notes

- All endpoints are scoped to `company_id` extracted from the JWT middleware context.
- The Vendor model uses GORM soft deletes (`deleted_at` timestamp).
- On create, a User record is inserted first; the resulting `user.id` is then assigned to `vendor.user_id`.
- Vendor email uniqueness is enforced per company via a composite database index (`idx_vendor_company_email`).
- The vendor list endpoint caps results at a maximum of 100 per page.
- Partial updates on PUT use a `map` type so only supplied fields are written; omitted fields are not overwritten.
- The linked user's role is hardcoded to `role_id = 4` (vendor portal role).

---

# Data Model

**Table: `vendors`**

| Column            | Type             | Constraints                                          | Notes                                       |
|-------------------|------------------|------------------------------------------------------|---------------------------------------------|
| `id`              | uint             | PK, Auto Increment                                   |                                             |
| `company_id`      | uint             | FK → `companies.id`, Indexed                         | Scoped to authenticated company             |
| `user_id`         | *uint (nullable) | FK → `users.id`                                      | Linked user account; optional               |
| `name`            | varchar(255)     | NOT NULL                                             |                                             |
| `email`           | varchar(255)     | Unique per company via `idx_vendor_company_email`    |                                             |
| `phone`           | varchar(50)      |                                                      |                                             |
| `address`         | string           |                                                      |                                             |
| `logo`            | string           |                                                      | File path or URL                            |
| `uploaded_by`     | *uint (nullable) | FK → `saas_users.id`                                 | Who uploaded the logo; JSON key: `uploadedBy` |
| `status`          | enum             | Default `'Active'`                                   | Values: `Active`, `Inactive`, `Blocked`     |
| `description`     | text             |                                                      |                                             |
| `total_paid`      | float64          | Default `0`                                          | JSON key: `totalPaid`                       |
| `amount_payable`  | float64          | Default `0`                                          | JSON key: `amountPayable`                   |
| `created_at`      | timestamp        | Auto-managed                                         | JSON key: `createdAt`                       |
| `updated_at`      | timestamp        | Auto-managed                                         | JSON key: `updatedAt`                       |
| `deleted_at`      | timestamp        | Soft delete; null if not deleted                     | JSON key: `"-"` (never returned)            |

**Relationships:**

| Relationship              | Type        | Key                     | Notes                                      |
|---------------------------|-------------|-------------------------|--------------------------------------------|
| Vendor → User             | BelongsTo   | `user_id`               | Optional; preloaded on single-record fetch |
| Vendor → User.Role        | Nested      | via User                | Preloaded via User association             |
| Vendor ← Products         | HasMany     | `Product.vendor_id`     | Managed by the products module             |
| Vendor ← VendorReturns    | HasMany     | `VendorReturn.vendor_id`| Managed by the vendor-returns module       |

---

# API Endpoints

| Method | URL             | Auth       | Purpose                                   |
|--------|-----------------|------------|-------------------------------------------|
| GET    | `/vendors/`     | Bearer JWT | List vendors with pagination and filtering |
| GET    | `/vendors/:id`  | Bearer JWT | Get a single vendor by ID                 |
| POST   | `/vendors/`     | Bearer JWT | Create a new vendor                       |
| PUT    | `/vendors/:id`  | Bearer JWT | Partially update a vendor                 |
| DELETE | `/vendors/:id`  | Bearer JWT | Soft-delete a vendor                      |

---

# Request Payloads

**GET `/vendors/` — Query Parameters**

| Parameter | Type   | Default | Max | Description                                                  |
|-----------|--------|---------|-----|--------------------------------------------------------------|
| `page`    | int    | `1`     | —   | Page number                                                  |
| `limit`   | int    | `10`    | 100 | Records per page                                             |
| `search`  | string | —       | —   | LIKE match on `name`, `email`, and `phone`                   |
| `status`  | string | —       | —   | Filter by `Active`, `Inactive`, `Blocked`, or `all` for no filter |

---

**POST `/vendors/` — Request Body**

```json
{
  "name":        "string (required)",
  "email":       "string (required)",
  "phone":       "string (optional)",
  "address":     "string (optional)",
  "status":      "string (optional) — Active | Inactive | Blocked",
  "description": "string (optional)"
}
```

---

**PUT `/vendors/:id` — Request Body**

Partial update. Any subset of vendor fields may be supplied; all fields are optional.

```json
{
  "name":          "string (optional)",
  "email":         "string (optional)",
  "phone":         "string (optional)",
  "address":       "string (optional)",
  "logo":          "string (optional)",
  "status":        "string (optional)",
  "description":   "string (optional)",
  "totalPaid":     "float64 (optional)",
  "amountPayable": "float64 (optional)"
}
```

---

# Response Contracts

**GET `/vendors/`** — 200 OK

```json
{
  "message": "Vendors retrieved successfully",
  "data": [
    {
      "id":            1,
      "companyId":     1,
      "userId":        6,
      "user": {
        "id":    6,
        "username": "Acme Supplies",
        "email": "acme@example.com",
        "role":  { "id": 4, "name": "Vendor" }
      },
      "name":          "Acme Supplies",
      "email":         "acme@example.com",
      "phone":         "+1234567890",
      "address":       "456 Supply Ave",
      "logo":          "",
      "status":        "Active",
      "description":   "",
      "totalPaid":     0,
      "amountPayable": 0,
      "createdAt":     "2024-01-01T00:00:00Z",
      "updatedAt":     "2024-01-01T00:00:00Z"
    }
  ],
  "total": 20,
  "page":  1,
  "limit": 10
}
```

---

**GET `/vendors/:id`** — 200 OK

Same shape as a single list element above, with `user` and `role` preloaded.

```json
{
  "message": "Vendor fetched successfully",
  "data": {
    "id":          1,
    "companyId":   1,
    "userId":      6,
    "name":        "Acme Supplies",
    "email":       "acme@example.com",
    "phone":       "+1234567890",
    "address":     "456 Supply Ave",
    "logo":        "",
    "status":      "Active",
    "description": "",
    "totalPaid":   0,
    "amountPayable": 0,
    "user": {
      "id":       6,
      "username": "Acme Supplies",
      "email":    "acme@example.com",
      "role":     { "id": 4, "name": "Vendor" }
    },
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

---

**POST `/vendors/`** — 201 Created

Returns the created vendor object (same shape as GET `/vendors/:id` data).

```json
{
  "message": "Vendor created successfully",
  "data": { }
}
```

---

**PUT `/vendors/:id`** — 200 OK

```json
{
  "message": "Vendor updated successfully",
  "data": { }
}
```

---

**DELETE `/vendors/:id`** — 200 OK

```json
{
  "message": "Vendor deleted successfully"
}
```

---

# Validation Rules

| Field    | Rule                                                                        |
|----------|-----------------------------------------------------------------------------|
| `name`   | Required on POST. Must be a non-empty string.                               |
| `email`  | Required on POST. Must be unique per `company_id` (enforced at DB level).   |
| `status` | If provided, must be one of: `Active`, `Inactive`, `Blocked`.               |
| `limit`  | Maximum value of 100 on the list endpoint.                                  |
| `page`   | Minimum 1; defaults to 1 if not provided.                                   |

---

# Business Logic Details

### Vendor Creation (POST `/vendors/`)

1. Extract `company_id` from JWT context.
2. Validate that `name` and `email` are present in the request body.
3. Hash the default password `"changeme"` using bcrypt.
4. Create a new record in the `users` table with:
   - `username` = vendor `name`
   - `email` = vendor `email`
   - `password` = bcrypt-hashed `"changeme"`
   - `role_id` = 4
5. Set `vendor.user_id` = newly created user's `id`.
6. Insert the vendor record.

### Vendor Update (PUT `/vendors/:id`)

- Uses a partial map update — only provided fields are written to the database.
- If `name` or `email` is included in the payload, the corresponding fields (`username` and/or `email`) are also updated on the linked `users` record.

### Vendor Deletion (DELETE `/vendors/:id`)

- Soft-deletes the vendor record (sets `deleted_at` on the vendor).
- Also soft-deletes the linked user record (sets `deleted_at` on the `users` row).

### Company Scoping

- All queries filter by `company_id` from the JWT. Vendors belonging to other companies are never visible or accessible.

---

# Dependencies

| Module / Table    | Relationship                                                                     |
|-------------------|----------------------------------------------------------------------------------|
| `users`           | Linked user created on vendor creation; synced on update; soft-deleted on vendor delete |
| `products`        | Products reference vendors via `vendor_id` (managed by the products module)      |
| `vendor-returns`  | VendorReturns belong to vendors via `vendor_id` (managed by the vendor-returns module) |
| `companies`       | `company_id` FK; all vendor records are scoped to a company                      |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is always read from the JWT context — clients cannot access or modify vendors belonging to other companies.
- Newly created vendor user accounts receive the default password `"changeme"`. It is expected (but not enforced within this module) that vendors change this password upon first login.
- Passwords are stored as bcrypt hashes; the plaintext value is never persisted.

---

# Error Handling

| HTTP Status | Trigger                                                                           |
|-------------|-----------------------------------------------------------------------------------|
| 400         | Missing required fields (`name`, `email`) on POST; malformed/invalid request body |
| 404         | Vendor not found for the given `:id` within the authenticated company             |
| 500         | Failed to hash password; failed to create linked user; failed to create or update vendor record |

**Error response shape:**

```json
{ "error": "message string" }
```

With optional `details` field for certain 400/500 errors:

```json
{ "error": "Invalid request body", "details": "..." }
```

---

# Testing Notes

- Verify that POST `/vendors/` creates a corresponding `users` record with `role_id = 4` and a bcrypt-hashed `"changeme"` password.
- Verify that updating a vendor's `name` or `email` propagates the change to the linked user record.
- Verify that DELETE soft-deletes both the vendor and the linked user (`deleted_at` is set on both rows).
- Test company scoping: a vendor belonging to Company A must not be accessible under Company B's JWT.
- Test the `idx_vendor_company_email` unique constraint: the same email cannot be used for two vendors within the same company but may be reused across companies.
- Test pagination: a `limit` value exceeding 100 should be capped at 100.
- Test `search` matching against `name`, `email`, and `phone` fields.
- Test each `status` filter value including `all` (no status filter applied).
- Verify that omitting optional fields in a PUT request does not overwrite existing values.

---

# Open Questions / Missing Details

- **Logo upload mechanism:** The `logo` field stores a string (URL or path). The upload process (S3, local storage, CDN) is not documented in this module.
- **Default password policy:** There is no documented enforcement or notification mechanism prompting vendors to change the default `"changeme"` password after first login.
- **`total_paid` / `amount_payable` update mechanism:** These fields default to `0` but it is unclear which module or process updates them (e.g., purchase orders, payment records).
- **Role ID 4:** The vendor portal role is hardcoded as `role_id = 4`. It is not confirmed whether this is a seeded constant or environment-dependent.
- **Vendor portal:** The linked user is created for "potential" portal access — it is not confirmed whether the vendor portal is fully implemented.
- **`logo` field in POST:** The `logo` field is present in the data model but is not listed in the POST request body. It is unclear if logo can be set at creation time or only via PUT.

---

# Change Log / Notes

| Date       | Note                                                                   |
|------------|------------------------------------------------------------------------|
| 2026-03-31 | Initial standardized documentation pass from source notes              |
