# Module: Users (Legacy)

## Business Purpose
Legacy user management for the non-SaaS part of the system. These users are referenced by customers, vendors, and staff as linked portal accounts. The user list is NOT scoped by company_id — it returns all users system-wide. This is distinct from SaaS users (`saas_users` table).

---

## Database Tables

### Table: `users`
| Column     | Type          | Notes                          |
|------------|---------------|--------------------------------|
| id         | uint (PK, AI) |                                |
| username   | varchar(100)  | json: username                 |
| email      | varchar(255)  | unique; json: email            |
| password   | varchar(255)  | bcrypt hashed; json: "-"       |
| role_id    | uint (FK)     | FK → roles.id; json: roleId    |
| address    | text          | optional; json: address        |
| created_at | timestamp     | json: createdAt                |
| updated_at | timestamp     | json: updatedAt                |
| deleted_at | timestamp     | Soft delete                    |

### Table: `roles`
| Column     | Type          | Notes             |
|------------|---------------|-------------------|
| id         | uint (PK, AI) |                   |
| title      | string        | json: title       |
| status     | bool          | default false     |
| created_at | timestamp     | json: createdAt   |
| updated_at | timestamp     | json: updatedAt   |
| deleted_at | timestamp     | Soft delete       |

---

## Relationships
- User → Role (BelongsTo via role_id)
- User ← Customer (via customer.user_id; role_id=3)
- User ← Vendor (via vendor.user_id; role_id=4)
- User ← Staff (via staff.user_id; role_id=5)

---

## Business Logic
- **No company scoping** — all queries are global (no company_id filter).
- Delete returns `204 No Content` (unlike other modules which return 200).
- `password` field is never returned in JSON responses (json:"-").

---

## Endpoints

### GET /users/
**Auth:** Bearer JWT required
**Purpose:** List all users (no company filter, no pagination). Users are preloaded with their Role.

**Response 200:**
```json
{
  "message": "Users retrieved successfully",
  "data": [
    {
      "id": 1,
      "username": "John Doe",
      "email": "john@example.com",
      "roleId": 3,
      "role": { "id": 3, "title": "Customer", "status": true },
      "address": "",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

---

### GET /users/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "User fetched successfully",
  "data": {
    "id": 1,
    "username": "John Doe",
    "email": "john@example.com",
    "roleId": 3,
    "role": { "id": 3, "title": "Customer", "status": true },
    "address": "",
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**Error:** `404` — `{"error": "User not found"}`

---

### POST /users/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "username": "John Doe",
  "email": "john@example.com",
  "password": "plaintext",
  "roleId": 3,
  "address": ""
}
```

**Response 201:**
```json
{
  "id": 1,
  "username": "John Doe",
  "email": "john@example.com",
  "roleId": 3,
  "address": "",
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

### PUT /users/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** Any subset of user fields.

**Response 200:** User object directly.

---

### DELETE /users/:id
**Auth:** Bearer JWT required

**Response 204 No Content** (empty body)

---

## Dependencies
- **customers** — linked user created on customer creation (role_id=3)
- **vendors** — linked user created on vendor creation (role_id=4)
- **staff** — linked user created on staff creation (role_id=5)
