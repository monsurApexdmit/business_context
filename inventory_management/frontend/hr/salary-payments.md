# Frontend Module: Salary Payments

## Pages
- `/dashboard/hr/salary` — Salary payment list and history
- `/dashboard/hr/salary/new` — Record a salary payment
- `/dashboard/hr/salary/:id` — Salary payment detail / edit

## API Service
`lib/staffApi.ts` — exported as `salaryPaymentApi`

---

## GET /salary-payments
**Purpose:** Fetch paginated list of salary payment records
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `staffId` — filter payments for a specific staff member (optional)
- `month` — filter by month string (optional, e.g., `"2024-01"`)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "staffId": 5,
      "staff": {
        "id": 5,
        "userId": 10,
        "name": "Jane Smith",
        "email": "jane@example.com",
        "contact": "+1234567890",
        "joiningDate": "2024-01-15T00:00:00Z",
        "role": "Sales Associate",
        "status": "Active",
        "published": true,
        "avatar": "string",
        "salary": 3500.00,
        "bankAccount": "string",
        "paymentMethod": "string",
        "createdAt": "2024-01-01T00:00:00Z",
        "updatedAt": "2024-01-01T00:00:00Z"
      },
      "month": "2024-01",
      "amount": 3500.00,
      "paidAmount": 3500.00,
      "status": "Paid | Pending | Partial",
      "paymentDate": "2024-01-31T00:00:00Z",
      "paymentMethod": "bank_transfer | cash",
      "notes": "string",
      "createdAt": "2024-01-31T00:00:00Z",
      "updatedAt": "2024-01-31T00:00:00Z"
    }
  ],
  "total": 48,
  "page": 1,
  "limit": 20
}
```

**Note:** Pagination fields `total`, `page`, `limit` are at top level. `staff` object is optionally embedded. `status` uses Title Case: `"Paid"`, `"Pending"`, or `"Partial"`.

**Frontend Impact:**
- Salary payment list filterable by staff member and month
- Status badges: Paid (green), Pending (yellow), Partial (orange)
- `paidAmount` vs `amount` shows partial payment progress

---

## GET /salary-payments/:id
**Purpose:** Fetch a single salary payment record
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric salary payment ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full SalaryPaymentResponse object with embedded staff..." }
}
```

**Frontend Impact:**
- Payment detail view with staff info
- Edit form pre-populated for adjustments

---

## POST /salary-payments
**Purpose:** Record a new salary payment
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "staffId": 5,
  "month": "2024-01",
  "amount": 3500.00,
  "paidAmount": 3500.00,
  "status": "Paid | Pending | Partial",
  "paymentDate": "2024-01-31T00:00:00Z",
  "paymentMethod": "bank_transfer",
  "notes": "January salary"
}
```

**Note:** `staffId`, `month`, `amount`, `paidAmount`, and `status` are required. `paymentDate`, `paymentMethod`, and `notes` are optional. `month` format is not strictly enforced in TypeScript — backend may expect `"YYYY-MM"` or `"YYYY-MM-DD"`.

**Expected Response 201:**
```json
{
  "message": "Salary payment recorded successfully",
  "data": { "...full SalaryPaymentResponse object..." }
}
```

**Frontend Impact:**
- New payment appears in salary list
- For partial payments: `status: "Partial"` with `paidAmount < amount`
- Multiple payments for same staff/month are allowed (to record partial payments over time)

---

## PUT /salary-payments/:id
**Purpose:** Update an existing salary payment record
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric salary payment ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "staffId": 5,
  "month": "2024-01",
  "amount": 3500.00,
  "paidAmount": 3500.00,
  "status": "Paid | Pending | Partial",
  "paymentDate": "2024-01-31T00:00:00Z",
  "paymentMethod": "bank_transfer",
  "notes": "Updated: full payment confirmed"
}
```

**Note:** This is a full PUT (all required fields must be sent). `UpdateSalaryPaymentData` has the same required fields as `CreateSalaryPaymentData`. All fields except `paymentDate`, `paymentMethod`, and `notes` are required.

**Expected Response 200:**
```json
{
  "message": "Salary payment updated successfully",
  "data": { "...full SalaryPaymentResponse object..." }
}
```

**Frontend Impact:**
- Updating a Partial payment to Paid marks the full payment as complete
- Payment history audit trail

---

## DELETE /salary-payments/:id
**Purpose:** Delete a salary payment record
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric salary payment ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Salary payment deleted successfully"
}
```

**Frontend Impact:**
- Payment record removed from history
- Use with caution — financial records should generally not be deleted (soft delete recommended)
