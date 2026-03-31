# Module Overview

**Module Name:** Salary Payments
**Module Path:** `inventory_management/backend/salary-payments`

This module tracks salary payment records for staff members. Each record represents a single staff member's salary for a given month. Payment status (Paid / Partial / Pending) is automatically derived from the relationship between the total salary amount and the amount actually paid. Records are scoped by `company_id`.

---

# Business Context

The Salary Payments module provides a simple payroll ledger at the company level. Managers can record monthly salary payments for each staff member, log partial payments as advances, and track overall payment status. The module does not handle payroll calculations, tax, or deductions — it records what has been paid against what is owed.

---

# Functional Scope

## Included

- Create, read, update, and delete salary payment records per company
- Automatic status calculation: `Paid`, `Partial`, or `Pending` based on `paidAmount` vs `amount`
- Filtering by staff member, month, and payment status
- Paginated list ordered by most recent first
- Staff relation preloaded on all responses
- One record per staff member per month enforced via unique constraint
- Scoped by `company_id` extracted from JWT

## Excluded

- Payroll calculation or gross/net pay computation
- Tax or deduction management
- Bulk operations

---

# Architecture Notes

- Soft delete is supported via `deleted_at`. Hard delete logic should be replaced with soft delete.
- Records are ordered by `created_at DESC` in all list responses.
- `status` is never set directly by the caller; it is always computed server-side from `amount` and `paidAmount`.
- The unique constraint on `(staff_id, month)` is enforced at the database level. Only one salary payment record is allowed per staff member per month.
- The `Staff` relation is preloaded on all responses (list, get by ID, create, update).

---

# Data Model

**Table:** `salary_payments`

| Column           | Type         | Constraints / Notes                                                    |
|------------------|--------------|------------------------------------------------------------------------|
| `id`             | uint         | Primary key, auto-increment                                            |
| `company_id`     | uint         | FK → `companies.id`; indexed                                           |
| `staff_id`       | uint         | FK → `staff.id`; not null                                              |
| `month`          | varchar(50)  | Not null; e.g., `"2024-01"` or `"January 2024"`                        |
| `amount`         | float64      | Total salary owed; not null; default `0`                               |
| `paid_amount`    | float64      | Amount actually paid; default `0`; JSON key: `paidAmount`              |
| `status`         | enum         | `Paid` / `Pending` / `Partial`; default `Pending`; auto-calculated     |
| `payment_date`   | varchar(50)  | JSON key: `paymentDate`                                                |
| `payment_method` | varchar(100) | JSON key: `paymentMethod`                                              |
| `notes`          | text         |                                                                        |
| `created_at`     | timestamp    | JSON key: `createdAt`; list ordered by this field DESC                 |
| `updated_at`     | timestamp    | JSON key: `updatedAt`                                                  |
| `deleted_at`     | timestamp    | Soft delete; JSON key omitted (`json:"-"`)                             |

**Unique Constraint:** `(staff_id, month)` — one record per staff member per month.

**Relationships:**

| Relationship              | Type       | Notes                                        |
|---------------------------|------------|----------------------------------------------|
| SalaryPayment → Staff     | BelongsTo  | Via `staff_id`; preloaded on all responses   |

---

# API Endpoints

| Method | Path                      | Auth       | Description                                          |
|--------|---------------------------|------------|------------------------------------------------------|
| GET    | `/salary-payments/`       | Bearer JWT | Paginated list with staff, month, and status filters |
| GET    | `/salary-payments/:id`    | Bearer JWT | Single salary payment with staff preloaded           |
| POST   | `/salary-payments/`       | Bearer JWT | Create a new salary payment record                   |
| PUT    | `/salary-payments/:id`    | Bearer JWT | Full update; status is auto-recalculated             |
| DELETE | `/salary-payments/:id`    | Bearer JWT | Permanently delete a salary payment record           |

---

# Request Payloads

### GET `/salary-payments/` — Query Parameters

| Parameter  | Type   | Default | Notes                                                     |
|------------|--------|---------|-----------------------------------------------------------|
| `page`     | int    | `1`     |                                                           |
| `limit`    | int    | `10`    | Maximum: `100`                                            |
| `staff_id` | uint   | —       | Filter records to a specific staff member                 |
| `month`    | string | —       | Filter by month string (exact match)                      |
| `status`   | string | —       | `Paid` / `Pending` / `Partial`; `"all"` disables filter   |

### POST `/salary-payments/` — Create Salary Payment

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

| Field           | Type    | Required | Notes                                                       |
|-----------------|---------|----------|-------------------------------------------------------------|
| `staffId`       | uint    | Yes      | Must reference an existing staff member in the company      |
| `month`         | string  | Yes      | e.g., `"2024-01"` or `"January 2024"`                       |
| `amount`        | float64 | Yes      | Total salary owed for the month                             |
| `paidAmount`    | float64 | No       | Amount paid so far; defaults to `0`                         |
| `paymentDate`   | string  | No       |                                                             |
| `paymentMethod` | string  | No       |                                                             |
| `notes`         | string  | No       |                                                             |

> `status` is not accepted from the client. It is always auto-calculated server-side.

### PUT `/salary-payments/:id` — Update Salary Payment

Same fields as POST. All fields are supplied (full update). Status is auto-recalculated from the new `paidAmount` and existing `amount`.

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

---

# Response Contracts

### GET `/salary-payments/` — `200 OK`

```json
{
  "message": "Salary payments retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "staffId": 2,
      "staff": {
        "id": 2,
        "name": "Jane Smith"
      },
      "month": "2024-01",
      "amount": 3000,
      "paidAmount": 3000,
      "status": "Paid",
      "paymentDate": "2024-01-31",
      "paymentMethod": "Bank Transfer",
      "notes": "",
      "createdAt": "2024-01-31T00:00:00Z",
      "updatedAt": "2024-01-31T00:00:00Z"
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 10
}
```

### GET `/salary-payments/:id` — `200 OK`

```json
{
  "message": "Salary payment fetched successfully",
  "data": { }
}
```

Returns salary payment with `staff` preloaded.

### POST `/salary-payments/` — `201 Created`

```json
{
  "message": "Salary payment created successfully",
  "data": { }
}
```

Returns the created record with `staff` preloaded.

### PUT `/salary-payments/:id` — `200 OK`

```json
{
  "message": "Salary payment updated successfully",
  "data": { }
}
```

Returns the updated record with `staff` preloaded.

### DELETE `/salary-payments/:id` — `200 OK`

```json
{
  "message": "Salary payment deleted successfully"
}
```

---

# Validation Rules

| Rule                                               | HTTP Response |
|----------------------------------------------------|---------------|
| `staffId` is required                              | `400`         |
| `month` is required                                | `400`         |
| `amount` is required                               | `400`         |
| `staffId` must reference an existing staff member  | `404`         |
| Only one record allowed per `(staff_id, month)`    | DB-level unique constraint (see Open Questions) |
| `limit` capped at `100`                            | Enforced server-side |

---

# Business Logic Details

**Status auto-calculation logic:**

| Condition                        | Resulting Status |
|----------------------------------|------------------|
| `paidAmount >= amount`           | `"Paid"`         |
| `paidAmount > 0` (but < amount)  | `"Partial"`      |
| `paidAmount == 0`                | `"Pending"`      |

- Status is calculated on both create and update. Callers cannot set `status` directly.
- On update: status is re-derived from the **new** `paidAmount` and the **updated** `amount` (since `amount` can also be changed on update).
- `company_id` is always injected from JWT middleware; it is never accepted from the client.
- Records are ordered by `created_at DESC` in all list responses.
- The `Staff` relation is always preloaded on create, update, get by ID, and list responses.
- Deletion is a soft delete via `deleted_at`. Records are retained for payroll audit history.

---

# Dependencies

| Module / Table | Direction    | Notes                                             |
|----------------|--------------|---------------------------------------------------|
| `companies`    | FK (inbound) | `company_id` scopes all salary payment records    |
| `staff`        | FK (inbound) | `staff_id` references the staff member being paid |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is extracted from the JWT and used to scope all queries; users cannot access salary records belonging to other companies.
- No role-based access control is documented at the salary payments module level (see Open Questions).
- Deletion uses soft delete (`deleted_at`); records are not permanently removed.

---

# Error Handling

| HTTP Status | Scenario                                                         |
|-------------|------------------------------------------------------------------|
| `400`       | Invalid or missing request body; missing required fields         |
| `401`       | Missing or invalid JWT token                                     |
| `404`       | Salary payment not found for `:id`; staff member not found       |
| `500`       | Unexpected server or database error                              |

**Error response examples:**

```json
{ "error": "Invalid request body", "details": "..." }
{ "error": "Staff not found" }
{ "error": "Salary payment not found" }
{ "error": "Failed to create salary payment" }
{ "error": "Failed to update salary payment" }
```

---

# Testing Notes

- Test status calculation for all three cases: `paidAmount = 0` (Pending), `0 < paidAmount < amount` (Partial), `paidAmount >= amount` (Paid).
- Test that `status` in the request body is ignored; only the calculated value is stored.
- Test the unique constraint: creating a second record for the same `(staff_id, month)` must be rejected.
- Test that providing a non-existent `staffId` returns `404`.
- Test all three status filter values on the list endpoint; test `"all"` returns unfiltered results.
- Test `staff_id` filter on list.
- Test `month` filter on list.
- Test pagination and `limit` cap at `100`.
- Test that deletion uses soft delete: a deleted record's `deleted_at` is set and it is excluded from list/get queries.
- Test list ordering: most recently created records must appear first.
- Test that `staff` is preloaded on create, update, get by ID, and list responses.
- Confirm cross-company isolation: salary records from another company must not be accessible.

---

# Open Questions / Missing Details

- **Duplicate `(staff_id, month)` error response:** The unique constraint is enforced at the database level, but no explicit HTTP error response is documented for this case. Confirm what status code and message are returned (likely `409` or `500` depending on error handling).
- **`amount` on update:** It is unclear whether `amount` (total salary) is allowed to change on update, or whether only `paidAmount` changes. The status re-calculation on update uses the "existing amount" per the source spec, but the full update payload includes `amount`. Clarify whether `amount` can be modified after creation.
- **`month` format enforcement:** The `month` field accepts any string (e.g., `"2024-01"` or `"January 2024"`). There is no enforced format. Inconsistent formats may cause the month filter to miss records. A standardized format should be defined.
- **Role-based access control:** No documentation on whether specific SaaS roles have differing permissions on salary payment endpoints.
- **Soft delete:** `deleted_at` column added; deletion is now soft delete, retaining records for payroll audit history.

---

# Change Log / Notes

- Initial documentation authored from source specification.
- Soft delete (`deleted_at`) added to `salary_payments` table; replaces previous hard delete behavior.
- `month` format is free-form; standardization is recommended.
