# Frontend Module: Staff

## Pages
- `/dashboard/hr/staff` — Staff member list
- `/dashboard/hr/staff/new` — Add new staff member
- `/dashboard/hr/staff/:id` — Staff member detail / edit

## API Service
`lib/staffApi.ts` — exported as `staffApi`

---

## GET /staff
**Purpose:** Fetch paginated list of staff members
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by name or email (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "userId": 10,
      "name": "Jane Smith",
      "email": "jane@example.com",
      "contact": "+1234567890",
      "joiningDate": "2024-01-15T00:00:00Z",
      "role": "Sales Associate",
      "status": "Active | Inactive",
      "published": true,
      "avatar": "string",
      "salary": 3500.00,
      "bankAccount": "****4242",
      "paymentMethod": "bank_transfer | cash",
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 12,
  "page": 1,
  "limit": 20
}
```

**Note:** Pagination fields `total`, `page`, `limit` are at top level of response (not nested in `pagination` object). `status` uses Title Case: `"Active"` or `"Inactive"`.

**Frontend Impact:**
- Staff list table with status filter
- `salary` field used in salary payments module
- `role` is a free-text string (not linked to staff-roles system directly at the `staffApi` level)

---

## GET /staff/:id
**Purpose:** Fetch a single staff member by ID
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric staff ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full StaffResponse object..." }
}
```

**Frontend Impact:**
- Staff detail/edit form pre-populated
- Salary history loaded from `/salary-payments?staffId=:id`

---

## POST /staff
**Purpose:** Create a new staff member
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "contact": "+1234567890",
  "joiningDate": "2024-01-15T00:00:00Z",
  "role": "Sales Associate",
  "status": "Active",
  "published": true,
  "avatar": "string",
  "salary": 3500.00,
  "bankAccount": "****4242",
  "paymentMethod": "bank_transfer"
}
```

**Note:** `name`, `email`, and `contact` are required. All other fields optional. `avatar` is a URL string (no file upload on this endpoint).

**Expected Response 201:**
```json
{
  "message": "Staff member created successfully",
  "data": { "...full StaffResponse object..." }
}
```

**Frontend Impact:**
- New staff member appears in list
- `userId` populated by backend (links to user auth record if applicable)

---

## PUT /staff/:id
**Purpose:** Update a staff member's details
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric staff ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "string",
  "email": "string",
  "contact": "string",
  "joiningDate": "string",
  "role": "string",
  "status": "Active | Inactive",
  "published": true,
  "avatar": "string",
  "salary": 3800.00,
  "bankAccount": "string",
  "paymentMethod": "string"
}
```

**Note:** All fields optional for partial update.

**Expected Response 200:**
```json
{
  "message": "Staff member updated successfully",
  "data": { "...full StaffResponse object..." }
}
```

**Frontend Impact:**
- Edit form submission
- Salary change here does NOT create a salary payment record — use salary payments API for that

---

## DELETE /staff/:id
**Purpose:** Delete a staff member
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric staff ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Staff member deleted successfully"
}
```

**Frontend Impact:**
- Staff member removed from list
- Salary payment records for this staff may be orphaned (backend-dependent)
- If staff is linked to a user account, authentication access may also be revoked
