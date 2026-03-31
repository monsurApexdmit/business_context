# Module Overview

**Module Name:** Staff Roles
**File Path:** `inventory_management/backend/staff-roles/staff-roles.md`
**Last Updated:** 2026-03-31

The Staff Roles module manages role definitions for company staff within the SaaS team management system. Each role defines a named access profile and can carry granular read, write, and delete permissions on a per-module basis. Roles are scoped per company and are referenced by user accounts (`SaasUser`) and invitations.

---

# Business Context

In a multi-user SaaS environment, different team members need different levels of access to system modules (e.g., products, orders, reports). The Staff Roles module allows companies to define custom roles — such as `"Manager"` or `"Viewer"` — and attach a precise set of permissions to each. These roles are then assigned to users and invitation flows, controlling what each team member can see and do within the application.

---

# Functional Scope

## Included

- CRUD operations for staff role definitions (per company).
- Granular permission management per role: read, write, and delete flags on a per-module basis.
- Full replacement of a role's permissions on create and update (no partial patch of individual permissions).
- Flat permission array returned on every role response for ease of consumption.
- Cascading deletion of `role_permissions` rows when a role is deleted.
- Company-scoped queries via JWT context.

## Excluded

- Enforcement of permissions at the API level (this module only defines permissions; enforcement is the responsibility of consuming modules and middleware).
- Creation or management of the `permissions` records themselves — these are pre-seeded at the system level.
- Assignment of roles to users (handled by the `saas-team` module).
- Assignment of roles to invitations (handled by the `invitations` module).

---

# Architecture Notes

- All endpoints are scoped to `company_id` extracted from the Bearer JWT.
- Role names must be unique per company, enforced by the database index `idx_staffrole_company_name`.
- Permissions are never returned via GORM's association preloading directly. Instead, they are always fetched and serialized manually into a flat array (`roleResponse`).
- On every create or update, the entire set of `role_permissions` rows for the role is deleted and re-inserted from the submitted permissions array. There is no partial patch at the permission level.
- If a submitted permission references a `name` that does not exist in the pre-seeded `permissions` table, that entry is silently skipped — no error is returned.
- Role deletion cascades to `role_permissions` via a database-level `CASCADE DELETE` constraint. SaasUser and Invitation references (`role_id`) are not cascade-deleted — those constraints are managed at their respective modules.
- Soft delete is used on `staff_roles` (via `deleted_at`). `role_permissions` rows are removed via CASCADE on hard deletion of the role record.

---

# Data Model

## Table: `staff_roles`

| Column     | Type                 | Constraints                                  | JSON Key  |
|------------|----------------------|----------------------------------------------|-----------|
| id         | uint                 | PK, Auto Increment                           | id        |
| company_id | uint                 | FK → companies.id, Index                     | companyId |
| name       | varchar(255)         | Not null; unique per company (idx_staffrole_company_name) | name |
| created_at | timestamp            | Auto-managed                                 | createdAt |
| updated_at | timestamp            | Auto-managed                                 | updatedAt |
| deleted_at | timestamp (nullable) | Soft delete                                  | —         |

## Table: `permissions`

| Column | Type         | Constraints       | Notes                                         |
|--------|--------------|-------------------|-----------------------------------------------|
| id     | uint         | PK, Auto Increment|                                               |
| name   | varchar(255) | Unique            | Module name (e.g., `"products"`, `"orders"`); pre-seeded; not managed via API |

## Table: `role_permissions`

| Column        | Type | Constraints                              | Notes                     |
|---------------|------|------------------------------------------|---------------------------|
| id            | uint | PK                                       |                           |
| role_id       | uint | FK → staff_roles.id, CASCADE DELETE      |                           |
| permission_id | uint | FK → permissions.id                      |                           |
| read          | bool |                                          | Read access flag          |
| write         | bool |                                          | Write access flag         |
| delete        | bool |                                          | Delete access flag        |

## Relationships

| From          | Relationship | To             | Notes                                                         |
|---------------|--------------|----------------|---------------------------------------------------------------|
| StaffRole     | HasMany      | RolePermissions| Via `role_id`; not returned directly (`json:"-"`)             |
| RolePermission| BelongsTo    | Permission     | Via `permission_id`; used to resolve module name              |
| SaasUser      | BelongsTo    | StaffRole      | Via `SaasUser.role_id`; managed by `saas-team` module         |
| Invitation    | BelongsTo    | StaffRole      | Via `Invitation.role_id`; managed by `invitations` module     |

---

# API Endpoints

| Method | URL                | Auth       | Purpose                              |
|--------|--------------------|------------|--------------------------------------|
| GET    | `/staff-roles/`    | Bearer JWT | List all roles for the company       |
| GET    | `/staff-roles/:id` | Bearer JWT | Get a single role by ID              |
| POST   | `/staff-roles/`    | Bearer JWT | Create a new role with permissions   |
| PUT    | `/staff-roles/:id` | Bearer JWT | Update role name and/or permissions  |
| DELETE | `/staff-roles/:id` | Bearer JWT | Delete a role                        |

---

# Request Payloads

## `POST /staff-roles/` — Request Body

**Content-Type:** `application/json`

| Field                    | Type    | Required | Description                                                                        |
|--------------------------|---------|----------|------------------------------------------------------------------------------------|
| name                     | string  | Yes      | Role name; must be unique within the company                                       |
| permissions              | array   | No       | Array of permission objects. If omitted, role is created with no permissions.      |
| permissions[].name       | string  | Yes      | Module name matching a pre-seeded `permissions` record. Unknown names are skipped. |
| permissions[].read       | bool    | No       | Grant read access to this module                                                   |
| permissions[].write      | bool    | No       | Grant write access to this module                                                  |
| permissions[].delete     | bool    | No       | Grant delete access to this module                                                 |

**Example:**
```json
{
  "name": "Manager",
  "permissions": [
    { "name": "products", "read": true, "write": true, "delete": false },
    { "name": "orders",   "read": true, "write": false, "delete": false }
  ]
}
```

---

## `PUT /staff-roles/:id` — Request Body

**Content-Type:** `application/json`

| Field                | Type   | Required | Description                                                                                  |
|----------------------|--------|----------|----------------------------------------------------------------------------------------------|
| name                 | string | No       | New role name; must be unique within the company                                             |
| permissions          | array  | No       | If included, replaces all existing permissions entirely. If omitted, permissions are unchanged. |
| permissions[].name   | string | Yes      | Module name matching a pre-seeded `permissions` record. Unknown names are skipped.           |
| permissions[].read   | bool   | No       | Grant read access                                                                            |
| permissions[].write  | bool   | No       | Grant write access                                                                           |
| permissions[].delete | bool   | No       | Grant delete access                                                                          |

**Example:**
```json
{
  "name": "Senior Manager",
  "permissions": [
    { "name": "products", "read": true, "write": true, "delete": true }
  ]
}
```

---

# Response Contracts

All role endpoints return a `roleResponse` object — not the raw model. Permissions are serialized as a flat array.

## `roleResponse` Shape

```json
{
  "id": 1,
  "name": "Manager",
  "permissions": [
    { "name": "products", "read": true,  "write": true,  "delete": false },
    { "name": "orders",   "read": true,  "write": false, "delete": false }
  ],
  "createdAt": "2026-01-01T00:00:00Z",
  "updatedAt": "2026-01-01T00:00:00Z"
}
```

## `GET /staff-roles/` — 200 OK

```json
{
  "message": "Roles retrieved successfully",
  "data": [
    { /* roleResponse */ },
    { /* roleResponse */ }
  ]
}
```

## `GET /staff-roles/:id` — 200 OK

```json
{
  "message": "Role fetched successfully",
  "data": { /* roleResponse */ }
}
```

## `POST /staff-roles/` — 201 Created

```json
{
  "message": "Role created successfully",
  "data": { /* roleResponse */ }
}
```

## `PUT /staff-roles/:id` — 200 OK

```json
{
  "message": "Role updated successfully",
  "data": { /* roleResponse */ }
}
```

## `DELETE /staff-roles/:id` — 200 OK

```json
{
  "message": "Role deleted successfully"
}
```

---

# Validation Rules

| Field              | Rule                                                                                          |
|--------------------|-----------------------------------------------------------------------------------------------|
| name               | Required on create; non-empty string; must be unique within the company                       |
| permissions        | Optional array. If provided, must be a valid JSON array.                                      |
| permissions[].name | Must match an existing pre-seeded `permissions` record name. Unknown names are silently skipped. |
| permissions[].read / write / delete | Boolean flags. Absence is treated as `false`.                          |

---

# Business Logic Details

## Permission Replacement on Create / Update

Permissions are never merged or patched incrementally. The full lifecycle is:

1. On **create**: all submitted permissions are resolved against the `permissions` table by name. Matching rows are inserted into `role_permissions`. Unknown names are skipped without error.
2. On **update**: if a `permissions` array is included in the request body, all existing `role_permissions` rows for the role are deleted first, then new rows are inserted from the submitted array. If the `permissions` key is absent from the request body, existing permissions are not modified.

## Role Deletion

- The `staff_roles` record is soft-deleted (sets `deleted_at`).
- All associated `role_permissions` rows are removed via `CASCADE DELETE` at the database level.
- `SaasUser` and `Invitation` records that reference this role's `role_id` are **not** cascade-deleted. Behavior of those modules when referencing a deleted role is not defined in this spec.

## Response Serialization

Permissions are always fetched explicitly and assembled into a flat array before being returned. GORM's `HasMany` association on `RolePermissions` is marked `json:"-"` and is never returned directly.

---

# Dependencies

| Module        | Direction | Notes                                                                             |
|---------------|-----------|-----------------------------------------------------------------------------------|
| `permissions` | Outbound  | Pre-seeded permission records are looked up by name on every create/update        |
| `companies`   | Outbound  | All records scoped to `company_id` from JWT                                       |
| `saas-team`   | Inbound   | `SaasUser.role_id` references `staff_roles.id`                                   |
| `invitations` | Inbound   | `Invitation.role_id` references `staff_roles.id`                                 |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` is extracted from the JWT and enforced on all queries. Roles from other companies are never accessible.
- A `401` error is returned if `company_id` cannot be found in the JWT context (rather than the standard `403`).
- No additional RBAC enforcement is documented at this module level — any authenticated user of the company can manage roles.

---

# Error Handling

| HTTP Status | Trigger                                                                      |
|-------------|------------------------------------------------------------------------------|
| 400         | `name` is missing or empty on create                                         |
| 400         | Invalid request body (malformed JSON)                                        |
| 401         | `company_id` not found in JWT context                                        |
| 404         | Role not found for the given `id` within the company                         |
| 500         | Failed to retrieve, create, or update roles (database error)                 |

**Error response shape:**
```json
{ "error": "Human-readable error message" }
```

**Notable:** The `401` here is returned when the JWT is structurally valid but `company_id` is missing from the token's claims — it is a contextual authorization failure, not an authentication failure.

---

# Testing Notes

- **Permission replacement on update:** Submitting a `permissions` array on `PUT` must fully replace all existing permissions. Verify old permissions not in the new array are removed.
- **Permission omission on update:** Omitting the `permissions` key from the `PUT` body must leave existing permissions unchanged.
- **Unknown permission names:** Including a `permissions[].name` that does not exist in the pre-seeded `permissions` table must be silently skipped — no error, and no row inserted for that entry.
- **Role name uniqueness:** Creating two roles with the same name within the same company must fail. The same name across different companies must succeed.
- **Cascade delete:** Deleting a role must remove all its `role_permissions` rows. Verify no orphan rows remain.
- **Flat permissions array:** Confirm every role response serializes permissions as a flat array, not nested associations.
- **Company scoping:** Confirm authenticated requests cannot retrieve or modify roles belonging to a different company.
- **`401` on missing company context:** Simulate a JWT without a valid `company_id` claim and confirm a `401` is returned on all endpoints.
- **Empty permissions array on create:** Create a role with `"permissions": []` and verify the role is created with no permissions and the response returns an empty permissions array.

---

# Open Questions / Missing Details

- **Pre-seeded permission names:** The full list of valid module names in the `permissions` table is not documented. This information is required for clients to know which names to submit.
- **Silent skip behavior:** Unknown permission names are silently skipped. It is unclear whether the API should instead return a `400` with the list of unrecognized names. This may be a source of subtle bugs if callers misspell module names.
- **Role deletion with active users:** The behavior when a role is deleted while `SaasUser` records still reference it via `role_id` is not documented. It is unclear whether this is blocked by a FK constraint or allowed (leaving orphan references).
- **`401` vs `403` distinction:** The module returns `401` when `company_id` is missing from context. This is semantically a `403 Forbidden` (authenticated but not authorized). Confirm whether this is intentional.
- **Role permission defaults:** When `read`, `write`, or `delete` are omitted from a permission object, it is assumed they default to `false`. This should be verified.
- **RBAC enforcement:** It is unknown which module or middleware enforces the permissions defined here at the API request level.

---

# Change Log / Notes

| Date       | Author | Note                                                        |
|------------|--------|-------------------------------------------------------------|
| 2026-03-31 | System | Initial standardized documentation created from source spec |
