# Module Overview

**Module Name:** Staff
**File Path:** `inventory_management/backend/staff/staff.md`
**Last Updated:** 2026-03-31

The Staff module manages employee records for a company within the inventory management system. Each staff member is linked to a system User account (with a fixed role of `role_id = 5`). The module supports creating, reading, updating, and deleting staff records, with cascading operations on the linked user account.

---

# Business Context

Companies need to manage their internal workforce — cashiers, managers, warehouse staff, and other roles. The Staff module provides a structured employee directory that ties each employee to a user account, enabling them to log into the system. Default credentials are issued at creation and are expected to be changed by the employee. Salary information and payment method preferences are also tracked per staff member. Staff records are scoped to a single company.

---

# Functional Scope

## Included

- CRUD operations for staff records (per company).
- Automatic creation of a linked user account on staff creation (`role_id = 5`, default password `"changeme"`).
- Cascading update of the linked user's `username` and `email` when the staff member's `name` or `email` changes.
- Cascading soft-delete of the linked user when the staff member is deleted.
- Filtering by status (`Active` / `Inactive`) and role label.
- Search by name, email, or contact using a LIKE query.
- Pagination support on the list endpoint.
- Salary and payment method fields per staff member.

## Excluded

- Password management or forced password reset flows (handled outside this module).
- Payroll processing (salary payments are managed by the `salary-payments` module).
- Role-based access control configuration (handled by the `staff-roles` module).
- Avatar/image upload logic (field stored as a string reference only).

---

# Architecture Notes

- All endpoints are scoped to `company_id` extracted from the Bearer JWT.
- The email uniqueness constraint (`idx_staff_company_email`) is enforced per company — the same email may exist across different companies but not within the same company.
- User creation on staff create uses a two-step process: insert the user record first, then assign `user_id` on the staff record.
- The `role` field on the `staff` table is a plain string label (e.g., `"Manager"`, `"Cashier"`) — it is not a foreign key to `staff_roles`. It is used as a descriptive tag for the employee's job function.
- The linked user's role is determined by `role_id = 5`, which is a fixed system-level role for staff accounts.
- Staff records support soft delete via `deleted_at`. The linked user is also soft-deleted in the same operation.

---

# Data Model

## Table: `staff`

| Column         | Type                                     | Constraints                              | JSON Key      |
|----------------|------------------------------------------|------------------------------------------|---------------|
| id             | uint                                     | PK, Auto Increment                       | id            |
| company_id     | uint                                     | FK → companies.id, Index                 | companyId     |
| user_id        | *uint (nullable)                         | FK → users.id                            | userId        |
| name           | varchar(255)                             | Not null                                 | name          |
| email          | varchar(255)                             | Unique per company (idx_staff_company_email) | email      |
| contact        | varchar(50)                              |                                          | contact       |
| joining_date   | string                                   |                                          | joiningDate   |
| role           | string                                   | Descriptive label (e.g., "Manager")      | role          |
| status         | enum('Active','Inactive')                | Default: `'Active'`                      | status        |
| published      | bool                                     | Default: `false`                         | published     |
| avatar         | string                                   |                                          | avatar        |
| uploaded_by    | *uint (nullable)                         | FK → saas_users.id                       | uploadedBy    |
| salary         | float64                                  | Default: `0`                             | salary        |
| bank_account   | string                                   |                                          | bankAccount   |
| payment_method | enum('Bank Transfer','Cash','Check')     |                                          | paymentMethod |
| created_at     | timestamp                                | Auto-managed                             | createdAt     |
| updated_at     | timestamp                                | Auto-managed                             | updatedAt     |
| deleted_at     | timestamp (nullable)                     | Soft delete                              | —             |

## Relationships

| From          | Relationship | To             | Notes                                               |
|---------------|--------------|----------------|-----------------------------------------------------|
| Staff         | BelongsTo    | User           | Via `user_id`; optional (nullable FK)               |
| Staff.User    | BelongsTo    | Role           | Role preloaded through the linked User              |
| SalaryPayment | BelongsTo    | Staff          | Via `SalaryPayment.staff_id`; owned by salary-payments module |

---

# API Endpoints

| Method | URL          | Auth       | Purpose                                    |
|--------|--------------|------------|--------------------------------------------|
| GET    | `/staff/`    | Bearer JWT | List staff with search, filter, pagination |
| GET    | `/staff/:id` | Bearer JWT | Get a single staff member by ID            |
| POST   | `/staff/`    | Bearer JWT | Create a new staff member                  |
| PUT    | `/staff/:id` | Bearer JWT | Partially update a staff member            |
| DELETE | `/staff/:id` | Bearer JWT | Soft-delete a staff member                 |

---

# Request Payloads

## `GET /staff/` — Query Parameters

| Parameter | Type   | Required | Default | Description                                              |
|-----------|--------|----------|---------|----------------------------------------------------------|
| page      | int    | No       | 1       | Page number                                              |
| limit     | int    | No       | 10      | Records per page; max 100                                |
| search    | string | No       | —       | LIKE match on `name`, `email`, or `contact`              |
| status    | string | No       | —       | `Active` or `Inactive`. Use `"all"` to skip this filter  |
| role      | string | No       | —       | Filter by role label string. Use `"all"` to skip         |

---

## `POST /staff/` — Request Body

**Content-Type:** `application/json`

| Field         | Type    | Required | Description                                              |
|---------------|---------|----------|----------------------------------------------------------|
| name          | string  | Yes      | Full name of the staff member                            |
| email         | string  | Yes      | Email address; must be unique within the company         |
| contact       | string  | No       | Phone or contact number                                  |
| joiningDate   | string  | No       | Date of joining (free-form string, e.g., `"2024-01-01"`) |
| role          | string  | No       | Descriptive role label (e.g., `"Cashier"`, `"Manager"`)  |
| status        | string  | No       | `Active` or `Inactive`; defaults to `Active`             |
| salary        | float64 | No       | Monthly salary amount                                    |
| paymentMethod | string  | No       | `Bank Transfer`, `Cash`, or `Check`                      |

---

## `PUT /staff/:id` — Request Body

**Content-Type:** `application/json`

Any subset of the fields listed above (including `bankAccount`, `avatar`, `published`). Partial updates are supported — only provided fields are updated.

---

# Response Contracts

## `GET /staff/` — 200 OK

```json
{
  "message": "Staff retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "userId": 7,
      "user": {
        "id": 7,
        "username": "Jane Smith",
        "email": "jane@example.com",
        "role": { "id": 5, "name": "Staff" }
      },
      "name": "Jane Smith",
      "email": "jane@example.com",
      "contact": "+1234567890",
      "joiningDate": "2024-01-01",
      "role": "Manager",
      "status": "Active",
      "published": true,
      "avatar": "",
      "salary": 3000,
      "bankAccount": "1234567890",
      "paymentMethod": "Bank Transfer",
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 10,
  "page": 1,
  "limit": 10
}
```

## `GET /staff/:id` — 200 OK

```json
{
  "message": "Staff fetched successfully",
  "data": { /* staff object with user and role preloaded */ }
}
```

## `POST /staff/` — 201 Created

```json
{
  "message": "Staff created successfully",
  "data": { /* staff object */ }
}
```

## `PUT /staff/:id` — 200 OK

```json
{
  "message": "Staff updated successfully",
  "data": { /* updated staff object */ }
}
```

## `DELETE /staff/:id` — 200 OK

```json
{
  "message": "Staff deleted successfully"
}
```

---

# Validation Rules

| Field         | Rule                                                                                    |
|---------------|-----------------------------------------------------------------------------------------|
| name          | Required on create; non-empty string                                                    |
| email         | Required on create; must be unique within the company (`idx_staff_company_email`)       |
| status        | If provided, must be `Active` or `Inactive`                                             |
| paymentMethod | If provided, must be one of: `Bank Transfer`, `Cash`, `Check`                           |
| limit         | Max 100 on list endpoint; values above 100 are capped                                   |

---

# Business Logic Details

## Staff Creation

1. Validates `name` and `email` are present.
2. Hashes the default password `"changeme"`.
3. Creates a `users` record with `username = name`, `email = email`, hashed password, and `role_id = 5`.
4. Creates the `staff` record with `user_id` set to the new user's ID.
5. Returns the newly created staff record.

If user creation fails (e.g., duplicate email at the user level), the staff record is not created and a `500` error is returned.

## Staff Update

- If `name` is changed, the linked user's `username` is updated to match.
- If `email` is changed, the linked user's `email` is updated to match.
- Other staff fields (e.g., `salary`, `role`, `status`) are updated only on the `staff` record.

## Staff Deletion

- The staff record is soft-deleted by setting `deleted_at`.
- The linked user account is also soft-deleted in the same operation.
- The staff member will no longer appear in list queries after deletion.

---

# Dependencies

| Module            | Direction | Notes                                                                        |
|-------------------|-----------|------------------------------------------------------------------------------|
| `users`           | Outbound  | A new `users` record is created on every staff creation; updated/deleted in sync with staff changes |
| `companies`       | Outbound  | All records are scoped to `company_id` from JWT                              |
| `salary-payments` | Inbound   | `SalaryPayment.staff_id` references `staff.id`; owned by the salary-payments module |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` is extracted from the JWT and enforced on all queries — staff from other companies are never accessible.
- The default password for new staff user accounts is `"changeme"` (stored as a bcrypt hash). Staff are expected to change this on first login; however, no forced password reset mechanism is enforced at this module level.
- No role-level access control (RBAC) beyond JWT authentication is documented for this module.

---

# Error Handling

| HTTP Status | Trigger                                                                       |
|-------------|-------------------------------------------------------------------------------|
| 400         | `name` or `email` missing on create                                           |
| 400         | Invalid request body (malformed JSON)                                         |
| 404         | Staff member not found for the given `id` within the company                  |
| 500         | Failed to hash password during user creation                                  |
| 500         | Failed to create the linked user account                                      |
| 500         | Failed to create or update the staff record                                   |

**Error response shape:**
```json
{ "error": "Human-readable error message", "details": "Optional detail string" }
```

---

# Testing Notes

- **User creation on staff create:** Verify that a `users` record is created alongside the staff record. Confirm `username`, `email`, `role_id = 5`, and hashed password `"changeme"` are set correctly.
- **User sync on update:** Changing staff `name` must update `users.username`; changing staff `email` must update `users.email`. Changing `salary` or `role` must not modify the user record.
- **Cascading soft delete:** Deleting a staff member must soft-delete both the `staff` row and the linked `users` row. Neither should appear in subsequent list queries.
- **Email uniqueness:** Attempting to create two staff members with the same email within the same company must fail. The same email across different companies must succeed.
- **Filtering:** Test `status=Active`, `status=Inactive`, `status=all`, and role-based filtering independently and in combination with `search`.
- **Search:** Verify LIKE search matches partial strings on `name`, `email`, and `contact`.
- **Pagination:** Verify `page` and `limit` parameters; confirm `limit` is capped at 100.
- **Company scoping:** Confirm requests cannot retrieve staff belonging to a different company.

---

# Open Questions / Missing Details

- **`published` field semantics:** The `published` boolean field defaults to `false`. Its business purpose (e.g., visible in a directory, active in a POS selection list) is not documented.
- **`avatar` field:** Stored as a string. The upload mechanism, storage location, and URL format are not documented.
- **`joiningDate` format:** Stored as a plain string. No format validation (e.g., ISO 8601) is documented.
- **Forced password reset:** There is no documented mechanism requiring staff to change the default `"changeme"` password at first login.
- **Duplicate email error at user level:** If the `users` table already has the submitted email (from another context), staff creation returns `500`. Consider whether a more specific `409 Conflict` would be appropriate.
- **RBAC beyond JWT:** It is unknown whether specific staff roles or permission flags restrict access to staff management endpoints beyond requiring a valid JWT.
- **Response of `POST /staff/`:** It is unclear whether the response includes the linked user and role preloaded, or just the raw staff object.

---

# Change Log / Notes

| Date       | Author | Note                                                        |
|------------|--------|-------------------------------------------------------------|
| 2026-03-31 | System | Initial standardized documentation created from source spec |
