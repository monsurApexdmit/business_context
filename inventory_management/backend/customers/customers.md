# Module: Customers

## Business Purpose
Manages customer records for a company. Each customer also gets a linked User record (role_id=3) for potential portal access. Supports filtering by status and type (retail/wholesale). On create, a linked user is also created with a default password ("changeme"). On delete, the linked user is also soft-deleted.

---

## Database Table: `customers`

| Column        | Type           | Notes                                                    |
|---------------|----------------|----------------------------------------------------------|
| id            | uint (PK, AI)  |                                                          |
| company_id    | uint (FK)      | FK → companies.id; indexed                              |
| user_id       | *uint (FK)     | FK → users.id; nullable (linked user account)           |
| name          | varchar(255)   | not null                                                 |
| email         | varchar(255)   | unique per company: idx_customer_company_email           |
| phone         | varchar(50)    |                                                          |
| address       | string         |                                                          |
| city          | string         |                                                          |
| state         | string         |                                                          |
| zip_code      | string         | json: zipCode                                            |
| country       | string         |                                                          |
| customer_type | enum           | 'retail', 'wholesale'; default 'retail'; json: customerType |
| status        | enum           | 'active', 'inactive'; default 'active'                  |
| notes         | text           |                                                          |
| store_credit  | float64        | default 0; json: storeCredit                            |
| created_at    | timestamp      | json: createdAt                                         |
| updated_at    | timestamp      | json: updatedAt                                         |
| deleted_at    | timestamp      | Soft delete (json: "-")                                 |

---

## Relationships
- Customer → User (BelongsTo via user_id, optional)
- Customer → User.Role (Preloaded via User)
- Customer ← Orders (Sell.customer_id)
- Customer ← ShippingAddresses (ShippingAddress.customer_id)
- Customer ← CustomerReturns (CustomerReturn.customer_id)
- Customer ← CouponUsages (CouponUsage.customer_id)

---

## Business Logic
- On **create**: a linked `users` record is created (username=name, email=email, password="changeme" hashed, role_id=3). Then customer.user_id is set to that user's ID.
- On **update**: if `name` or `email` is changed in the customer, the linked user's `username`/`email` is also updated.
- On **delete**: customer is soft-deleted; linked user is also soft-deleted.
- Scoped by `company_id` from JWT context.

---

## Endpoints

### GET /customers/
**Auth:** Bearer JWT required
**Purpose:** List customers with search and filtering.

**Query Parameters:**
| Param  | Type   | Default | Description                              |
|--------|--------|---------|------------------------------------------|
| page   | int    | 1       |                                          |
| limit  | int    | 10      | max 100                                  |
| search | string |         | LIKE match on name, email, phone         |
| status | string |         | active / inactive; "all" = no filter     |
| type   | string |         | retail / wholesale; "all" = no filter    |

**Response 200:**
```json
{
  "message": "Customers retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "userId": 5,
      "user": { "id": 5, "username": "John Doe", "email": "john@example.com", "role": {...} },
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+1234567890",
      "address": "123 Main St",
      "city": "New York",
      "state": "NY",
      "zipCode": "10001",
      "country": "US",
      "customerType": "retail",
      "status": "active",
      "notes": "",
      "storeCredit": 0,
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "total": 50,
  "page": 1,
  "limit": 10
}
```

---

### GET /customers/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Customer fetched successfully",
  "data": { /* customer object with user and role */ }
}
```

**Error:** `404` — `{"error": "Customer not found"}`

---

### POST /customers/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+1234567890",
  "address": "123 Main St",
  "city": "New York",
  "state": "NY",
  "zipCode": "10001",
  "country": "US",
  "customerType": "retail",
  "status": "active",
  "notes": "",
  "storeCredit": 0
}
```

**Validation:**
- `name` — required
- `email` — required

**Response 201:**
```json
{
  "message": "Customer created successfully",
  "data": { /* customer object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Name and email are required"}`
- `500` — `{"error": "Failed to create user"}` / `{"error": "Failed to create customer"}`

---

### PUT /customers/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Partial update — accepts any customer fields as a map.

**Request Body:** Any subset of customer fields.

**Response 200:**
```json
{
  "message": "Customer updated successfully",
  "data": { /* customer object */ }
}
```

---

### DELETE /customers/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Customer deleted successfully"
}
```

---

## Dependencies
- **users** — linked user created on customer creation
- **orders (sells)** — customers referenced by sell.customer_id
- **shipping-addresses** — shipping addresses belong to customers
