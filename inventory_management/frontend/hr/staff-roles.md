# Frontend Module: Staff Roles

## Pages
- `/dashboard/hr/roles` — Staff role list with permissions
- `/dashboard/hr/roles/new` — Create role with permissions matrix
- `/dashboard/hr/roles/:id` — Edit role and permissions

## API Service
`lib/staffApi.ts` — exported as `staffRoleApi`

---

## GET /staff-roles
**Purpose:** Fetch paginated list of staff roles with their permission sets
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by role name (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "name": "Sales Associate",
      "permissions": [
        {
          "name": "products",
          "read": true,
          "write": false,
          "delete": false
        },
        {
          "name": "orders",
          "read": true,
          "write": true,
          "delete": false
        }
      ],
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 20
}
```

**Note:** Pagination fields `total`, `page`, `limit` are at top level of response (not nested). `permissions` is an array of resource permission objects with explicit `read`, `write`, `delete` booleans.

**Frontend Impact:**
- Role list table
- Permission matrix shows read/write/delete columns per resource
- Roles available in team invite form (`roleId` field)

---

## GET /staff-roles/:id
**Purpose:** Fetch a single staff role with its full permission set
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric staff role ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "name": "Sales Associate",
    "permissions": [
      { "name": "products", "read": true, "write": false, "delete": false },
      { "name": "orders", "read": true, "write": true, "delete": false }
    ],
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Edit role form with permissions checkboxes pre-populated

---

## POST /staff-roles
**Purpose:** Create a new staff role with permissions
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "Sales Associate",
  "permissions": [
    { "name": "products", "read": true, "write": false, "delete": false },
    { "name": "orders", "read": true, "write": true, "delete": false },
    { "name": "customers", "read": true, "write": true, "delete": false },
    { "name": "inventory", "read": true, "write": false, "delete": false }
  ]
}
```

**Note:** `name` and `permissions` are both required. `permissions` array should include all resource entries — omitted resources default to no permissions (backend-dependent behavior).

**Expected Response 201:**
```json
{
  "message": "Staff role created successfully",
  "data": { "...full StaffRoleResponse object..." }
}
```

**Frontend Impact:**
- New role available in team user invite form's role dropdown
- New role available in company team role assignment
- Role list refreshed after creation

---

## PUT /staff-roles/:id
**Purpose:** Update a staff role and its permissions (full replace)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric staff role ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "Senior Sales Associate",
  "permissions": [
    { "name": "products", "read": true, "write": true, "delete": false },
    { "name": "orders", "read": true, "write": true, "delete": true }
  ]
}
```

**Note:** Both `name` and `permissions` are required (full update, not partial). This is a PUT (full replace), so the entire permissions array must be sent.

**Expected Response 200:**
```json
{
  "message": "Staff role updated successfully",
  "data": { "...full StaffRoleResponse object..." }
}
```

**Frontend Impact:**
- Permission changes take effect for all users with this role immediately
- Existing sessions may need refresh to reflect new permission set (frontend-side enforcement may be needed)

---

## DELETE /staff-roles/:id
**Purpose:** Delete a staff role
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric staff role ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Staff role deleted successfully"
}
```

**Frontend Impact:**
- Role removed from list
- Users/staff assigned to this role may lose their role assignment (backend-dependent)
- Should prevent deletion if role is in use (client-side warning recommended)
