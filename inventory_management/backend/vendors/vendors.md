# Module: Vendors

## Business Purpose
Manages vendor (supplier) records for a company. Each vendor also gets a linked User record (role_id=4) for potential portal access. Supports filtering by status. On create, a linked user is also created with a default password ("changeme"). On delete, the linked user is also soft-deleted.

---

## Database Table: `vendors`

| Column        | Type           | Notes                                                      |
|---------------|----------------|------------------------------------------------------------|
| id            | uint (PK, AI)  |                                                            |
| company_id    | uint (FK)      | FK → companies.id; indexed                                 |
| user_id       | *uint (FK)     | FK → users.id; nullable (linked user account)              |
| name          | varchar(255)   | not null                                                   |
| email         | varchar(255)   | unique per company: idx_vendor_company_email               |
| phone         | varchar(50)    |                                                            |
| address       | string         |                                                            |
| logo          | string         |                                                            |
| status        | enum           | 'Active', 'Inactive', 'Blocked'; default 'Active'          |
| description   | text           |                                                            |
| total_paid    | float64        | default 0; json: totalPaid                                 |
| amount_payable| float64        | default 0; json: amountPayable                             |
| created_at    | timestamp      | json: createdAt                                            |
| updated_at    | timestamp      | json: updatedAt                                            |
| deleted_at    | timestamp      | Soft delete (json: "-")                                    |

---

## Relationships
- Vendor → User (BelongsTo via user_id, optional)
- Vendor → User.Role (Preloaded via User)
- Vendor ← Products (Product.vendor_id)
- Vendor ← VendorReturns (VendorReturn.vendor_id)

---

## Business Logic
- On **create**: a linked `users` record is created (username=name, email=email, password="changeme" hashed, role_id=4). Then vendor.user_id is set to that user's ID.
- On **update**: if `name` or `email` is changed in the vendor, the linked user's `username`/`email` is also updated.
- On **delete**: vendor is soft-deleted; linked user is also soft-deleted.
- Scoped by `company_id` from JWT context.

---

## Endpoints

### GET /vendors/
**Auth:** Bearer JWT required
**Purpose:** List vendors with search and filtering.

**Query Parameters:**
| Param  | Type   | Default | Description                              |
|--------|--------|---------|------------------------------------------|
| page   | int    | 1       |                                          |
| limit  | int    | 10      | max 100                                  |
| search | string |         | LIKE match on name, email, phone         |
| status | string |         | Active / Inactive / Blocked; "all" = no filter |

**Response 200:**
```json
{
  "message": "Vendors retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "userId": 6,
      "user": { "id": 6, "username": "Acme Supplies", "email": "acme@example.com", "role": {...} },
      "name": "Acme Supplies",
      "email": "acme@example.com",
      "phone": "+1234567890",
      "address": "456 Supply Ave",
      "logo": "",
      "status": "Active",
      "description": "",
      "totalPaid": 0,
      "amountPayable": 0,
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "total": 20,
  "page": 1,
  "limit": 10
}
```

---

### GET /vendors/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Vendor fetched successfully",
  "data": { /* vendor object with user and role */ }
}
```

**Error:** `404` — `{"error": "Vendor not found"}`

---

### POST /vendors/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "name": "Acme Supplies",
  "email": "acme@example.com",
  "phone": "+1234567890",
  "address": "456 Supply Ave",
  "status": "Active",
  "description": ""
}
```

**Validation:**
- `name` — required
- `email` — required

**Response 201:**
```json
{
  "message": "Vendor created successfully",
  "data": { /* vendor object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Name and email are required"}`
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `500` — `{"error": "Failed to process password"}` / `{"error": "Failed to create user", "details": "..."}` / `{"error": "Failed to create vendor"}`

---

### PUT /vendors/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Partial update — accepts any vendor fields as a map.

**Request Body:** Any subset of vendor fields.

**Response 200:**
```json
{
  "message": "Vendor updated successfully",
  "data": { /* vendor object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body"}`
- `404` — `{"error": "Vendor not found"}`
- `500` — `{"error": "Failed to update vendor"}`

---

### DELETE /vendors/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Vendor deleted successfully"
}
```

**Error:** `404` — `{"error": "Vendor not found"}`

---

## Dependencies
- **users** — linked user created on vendor creation
- **products** — products can reference vendors via vendor_id
- **vendor-returns** — vendor returns belong to vendors
