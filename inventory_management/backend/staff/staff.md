# Module: Staff

## Business Purpose
Manages staff (employee) records for a company. Each staff member also gets a linked User record (role_id=5). Supports filtering by status and role. On create, a linked user is created with a default password ("changeme"). On delete, the linked user is also soft-deleted.

---

## Database Table: `staff`

| Column         | Type           | Notes                                                          |
|----------------|----------------|----------------------------------------------------------------|
| id             | uint (PK, AI)  |                                                                |
| company_id     | uint (FK)      | FK → companies.id; indexed                                     |
| user_id        | *uint (FK)     | FK → users.id; nullable (linked user account)                  |
| name           | varchar(255)   | not null                                                       |
| email          | varchar(255)   | unique per company: idx_staff_company_email                    |
| contact        | varchar(50)    | json: contact                                                  |
| joining_date   | string         | json: joiningDate                                              |
| role           | string         | Role label (e.g. "Manager", "Cashier")                         |
| status         | enum           | 'Active', 'Inactive'; default 'Active'; json: status           |
| published      | bool           | default false; json: published                                 |
| avatar         | string         | json: avatar                                                   |
| salary         | float64        | default 0; json: salary                                        |
| bank_account   | string         | json: bankAccount                                              |
| payment_method | enum           | 'Bank Transfer', 'Cash', 'Check'; json: paymentMethod          |
| created_at     | timestamp      | json: createdAt                                                |
| updated_at     | timestamp      | json: updatedAt                                                |
| deleted_at     | timestamp      | Soft delete (json: "-")                                        |

---

## Relationships
- Staff → User (BelongsTo via user_id, optional)
- Staff → User.Role (Preloaded via User)
- Staff ← SalaryPayments (SalaryPayment.staff_id)

---

## Business Logic
- On **create**: a linked `users` record is created (username=name, email=email, password="changeme" hashed, role_id=5). Then staff.user_id is set to that user's ID.
- On **update**: if `name` or `email` is changed in the staff, the linked user's `username`/`email` is also updated.
- On **delete**: staff is soft-deleted; linked user is also soft-deleted.
- Scoped by `company_id` from JWT context.

---

## Endpoints

### GET /staff/
**Auth:** Bearer JWT required
**Purpose:** List staff with search and filtering.

**Query Parameters:**
| Param  | Type   | Default | Description                                     |
|--------|--------|---------|--------------------------------------------------|
| page   | int    | 1       |                                                  |
| limit  | int    | 10      | max 100                                          |
| search | string |         | LIKE match on name, email, contact               |
| status | string |         | Active / Inactive; "all" = no filter             |
| role   | string |         | filter by role string; "all" = no filter         |

**Response 200:**
```json
{
  "message": "Staff retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "userId": 7,
      "user": { "id": 7, "username": "Jane Smith", "email": "jane@example.com", "role": {...} },
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
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "total": 10,
  "page": 1,
  "limit": 10
}
```

---

### GET /staff/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Staff fetched successfully",
  "data": { /* staff object with user and role */ }
}
```

**Error:** `404` — `{"error": "Staff not found"}`

---

### POST /staff/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "contact": "+1234567890",
  "joiningDate": "2024-01-01",
  "role": "Manager",
  "status": "Active",
  "salary": 3000,
  "paymentMethod": "Bank Transfer"
}
```

**Validation:**
- `name` — required
- `email` — required

**Response 201:**
```json
{
  "message": "Staff created successfully",
  "data": { /* staff object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Name and email are required"}`
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `500` — `{"error": "Failed to process password"}` / `{"error": "Failed to create user", "details": "..."}` / `{"error": "Failed to create staff"}`

---

### PUT /staff/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Partial update — accepts any staff fields as a map.

**Request Body:** Any subset of staff fields.

**Response 200:**
```json
{
  "message": "Staff updated successfully",
  "data": { /* staff object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body"}`
- `404` — `{"error": "Staff not found"}`
- `500` — `{"error": "Failed to update staff"}`

---

### DELETE /staff/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Staff deleted successfully"
}
```

**Error:** `404` — `{"error": "Staff not found"}`

---

## Dependencies
- **users** — linked user created on staff creation
- **salary-payments** — salary payment records belong to staff members
