# Module: Staff Roles

## Business Purpose
Manages role definitions for company staff (used in the SaaS team management system). Each role can have granular read/write/delete permissions per module. Roles are scoped per company. The permissions are managed via a join table (`role_permissions`) linking roles to permission records.

---

## Database Tables

### Table: `staff_roles`
| Column     | Type          | Notes                                                        |
|------------|---------------|--------------------------------------------------------------|
| id         | uint (PK, AI) |                                                              |
| company_id | uint (FK)     | FK → companies.id; indexed                                   |
| name       | varchar(255)  | not null; unique per company: idx_staffrole_company_name     |
| created_at | timestamp     | json: createdAt                                              |
| updated_at | timestamp     | json: updatedAt                                              |
| deleted_at | timestamp     | Soft delete (json: "-")                                      |

### Table: `permissions`
| Column | Type          | Notes              |
|--------|---------------|--------------------|
| id     | uint (PK, AI) |                    |
| name   | varchar(255)  | unique; module name |

### Table: `role_permissions`
| Column        | Type    | Notes                              |
|---------------|---------|------------------------------------|
| id            | uint PK |                                    |
| role_id       | uint FK | FK → staff_roles.id; CASCADE DELETE |
| permission_id | uint FK | FK → permissions.id                |
| read          | bool    |                                    |
| write         | bool    |                                    |
| delete        | bool    |                                    |

---

## Relationships
- StaffRole → RolePermissions (HasMany via role_id; `json:"-"` — not returned directly)
- RolePermission → Permission (BelongsTo via permission_id)
- StaffRole ← SaasUser (SaasUser.role_id)
- StaffRole ← Invitation (Invitation.role_id)

---

## Business Logic
- Permissions are always returned as a flat array on every role response (never nested via GORM associations directly).
- On **create/update**, permissions are replaced entirely: all existing `role_permissions` rows for the role are deleted, then new rows are inserted based on the submitted permissions array.
- Unknown permission names in the submitted array are silently skipped.
- On role **delete**, role_permissions rows are removed by CASCADE.
- Scoped by `company_id` from JWT context.

---

## Response Shape

All role endpoints return a `roleResponse` object (not the raw model):

```json
{
  "id": 1,
  "name": "Manager",
  "permissions": [
    { "name": "products", "read": true, "write": true, "delete": false },
    { "name": "orders", "read": true, "write": false, "delete": false }
  ],
  "createdAt": "2024-01-01T00:00:00Z",
  "updatedAt": "2024-01-01T00:00:00Z"
}
```

---

## Endpoints

### GET /staff-roles/
**Auth:** Bearer JWT required
**Purpose:** List all roles for the company with their permissions.

**Response 200:**
```json
{
  "message": "Roles retrieved successfully",
  "data": [
    {
      "id": 1,
      "name": "Manager",
      "permissions": [
        { "name": "products", "read": true, "write": true, "delete": false }
      ],
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

**Error Responses:**
- `401` — `{"error": "Company ID not found in context"}`
- `500` — `{"error": "Failed to retrieve roles"}`

---

### GET /staff-roles/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Role fetched successfully",
  "data": { /* roleResponse object */ }
}
```

**Error Responses:**
- `401` — `{"error": "Company ID not found in context"}`
- `404` — `{"error": "Role not found"}`

---

### POST /staff-roles/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "name": "Manager",
  "permissions": [
    { "name": "products", "read": true, "write": true, "delete": false },
    { "name": "orders", "read": true, "write": false, "delete": false }
  ]
}
```

**Validation:**
- `name` — required

**Response 201:**
```json
{
  "message": "Role created successfully",
  "data": { /* roleResponse object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Role name is required"}`
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `401` — `{"error": "Company ID not found in context"}`
- `500` — `{"error": "Failed to create role"}`

---

### PUT /staff-roles/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Update role name and/or permissions. If `permissions` is included, it replaces all existing permissions.

**Request Body:**
```json
{
  "name": "Senior Manager",
  "permissions": [
    { "name": "products", "read": true, "write": true, "delete": true }
  ]
}
```

**Response 200:**
```json
{
  "message": "Role updated successfully",
  "data": { /* roleResponse object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body"}`
- `401` — `{"error": "Company ID not found in context"}`
- `404` — `{"error": "Role not found"}`
- `500` — `{"error": "Failed to update role"}`

---

### DELETE /staff-roles/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Role deleted successfully"
}
```

**Error Responses:**
- `401` — `{"error": "Company ID not found in context"}`
- `404` — `{"error": "Role not found"}`

---

## Dependencies
- **permissions** — pre-seeded permission records referenced by name
- **saas-team** — SaasUser.role_id references staff_roles
- **invitations** — Invitation.role_id references staff_roles
