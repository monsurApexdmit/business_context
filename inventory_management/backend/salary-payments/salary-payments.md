# Module: Salary Payments

## Business Purpose
Tracks salary payment records for staff members. Each record represents a single staff member's salary for a given month. The status (Paid/Partial/Pending) is automatically calculated from the paid amount vs total amount. Scoped by company_id from JWT.

---

## Database Table: `salary_payments`

| Column         | Type           | Notes                                                    |
|----------------|----------------|----------------------------------------------------------|
| id             | uint (PK, AI)  |                                                          |
| company_id     | uint (FK)      | FK → companies.id; indexed                               |
| staff_id       | uint (FK)      | FK → staff.id; not null                                  |
| month          | varchar(50)    | not null (e.g. "2024-01", "January 2024")                |
| amount         | float64        | Total salary amount; not null; default 0                 |
| paid_amount    | float64        | Amount actually paid; default 0; json: paidAmount        |
| status         | enum           | 'Paid', 'Pending', 'Partial'; default 'Pending'          |
| payment_date   | varchar(50)    | json: paymentDate                                        |
| payment_method | varchar(100)   | json: paymentMethod                                      |
| notes          | text           |                                                          |
| created_at     | timestamp      | json: createdAt; ordered DESC by default                 |
| updated_at     | timestamp      | json: updatedAt                                          |

> Note: No soft delete on this table.

---

## Relationships
- SalaryPayment → Staff (BelongsTo via staff_id)

---

## Business Logic
- **Status calculation**: automatically derived from amount and paid_amount:
  - `paid_amount >= amount` → `"Paid"`
  - `paid_amount > 0` → `"Partial"`
  - else → `"Pending"`
- On **create**: status is auto-calculated from the submitted amount and paidAmount.
- On **update**: status is auto-recalculated from the new paidAmount vs the existing amount.
- Unique constraint on `(staff_id, month)` — one record per staff per month.
- `Staff` relation is preloaded on all responses.
- Ordered by `created_at DESC`.

---

## Endpoints

### GET /salary-payments/
**Auth:** Bearer JWT required
**Purpose:** List salary payments with filters.

**Query Parameters:**
| Param    | Type   | Default | Description                         |
|----------|--------|---------|-------------------------------------|
| page     | int    | 1       |                                     |
| limit    | int    | 10      | max 100                             |
| staff_id | uint   |         | Filter by staff member              |
| month    | string |         | Filter by month string              |
| status   | string |         | Paid / Pending / Partial; "all" = no filter |

**Response 200:**
```json
{
  "message": "Salary payments retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "staffId": 2,
      "staff": { "id": 2, "name": "Jane Smith", ... },
      "month": "2024-01",
      "amount": 3000,
      "paidAmount": 3000,
      "status": "Paid",
      "paymentDate": "2024-01-31",
      "paymentMethod": "Bank Transfer",
      "notes": "",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 10
}
```

---

### GET /salary-payments/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Salary payment fetched successfully",
  "data": { /* salary payment with staff preloaded */ }
}
```

**Error:** `404` — `{"error": "Salary payment not found"}`

---

### POST /salary-payments/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "staffId": 2,
  "month": "2024-01",
  "amount": 3000,
  "paidAmount": 1500,
  "paymentDate": "2024-01-15",
  "paymentMethod": "Bank Transfer",
  "notes": "Advance payment"
}
```

**Validation:**
- `staffId` — required
- `month` — required
- `amount` — required

**Response 201:**
```json
{
  "message": "Salary payment created successfully",
  "data": { /* salary payment with staff preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `404` — `{"error": "Staff not found"}`
- `500` — `{"error": "Failed to create salary payment"}`

---

### PUT /salary-payments/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Update payment info — status is auto-recalculated from new paidAmount.

**Request Body:**
```json
{
  "staffId": 2,
  "month": "2024-01",
  "amount": 3000,
  "paidAmount": 3000,
  "paymentDate": "2024-01-31",
  "paymentMethod": "Bank Transfer",
  "notes": "Full payment"
}
```

**Response 200:**
```json
{
  "message": "Salary payment updated successfully",
  "data": { /* salary payment with staff preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `404` — `{"error": "Salary payment not found"}`
- `500` — `{"error": "Failed to update salary payment"}`

---

### DELETE /salary-payments/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Salary payment deleted successfully"
}
```

**Error:** `404` — `{"error": "Salary payment not found"}`

---

## Dependencies
- **staff** — salary payments reference staff members
