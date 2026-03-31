# Frontend Module: Vendors

## Pages
- `/dashboard/procurement/vendors` — Vendor list management
- `/dashboard/procurement/vendors/new` — Create vendor form
- `/dashboard/procurement/vendors/:id` — Vendor detail / edit form

## API Service
`lib/vendorApi.ts`

---

## GET /vendors/
**Purpose:** Fetch paginated list of vendors
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by vendor name or email (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "name": "Acme Supplies",
      "email": "contact@acme.com",
      "phone": "+1234567890",
      "address": "456 Industrial Blvd",
      "logo": "string",
      "status": "active | inactive",
      "description": "string",
      "totalPaid": 5000.00,
      "amountPayable": 1200.00,
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "deleted_at": null
    }
  ],
  "pagination": {
    "total": 25,
    "page": 1,
    "limit": 20,
    "total_pages": 2,
    "has_next": true,
    "has_previous": false
  }
}
```

**Note:** `totalPaid` and `amountPayable` are camelCase in TypeScript but the backend response uses these exact keys. `created_at`/`updated_at` are snake_case (unlike most other modules which use camelCase).

**Frontend Impact:**
- Vendor list table showing outstanding payable amounts
- `amountPayable` shown as a financial summary per vendor
- `status` badge shows active/inactive

---

## GET /vendors/:id
**Purpose:** Fetch a single vendor by ID
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric vendor ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full VendorResponse object..." }
}
```

**Frontend Impact:**
- Vendor detail/edit form pre-populated
- Vendor profile shows total paid and outstanding amount

---

## POST /vendors/
**Purpose:** Create a new vendor
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "Acme Supplies",
  "email": "contact@acme.com",
  "phone": "+1234567890",
  "address": "456 Industrial Blvd",
  "logo": "string",
  "status": "active",
  "description": "string",
  "totalPaid": 0.00,
  "amountPayable": 0.00
}
```

**Note:** `name`, `email`, and `phone` are required. All other fields optional. `logo` is a URL string (not a file upload — image upload handled separately if needed).

**Expected Response 201:**
```json
{
  "message": "Vendor created successfully",
  "data": { "...full VendorResponse object..." }
}
```

**Frontend Impact:**
- Vendor appears in product creation form vendor dropdown after creation
- Vendor list refreshed

---

## PUT /vendors/:id
**Purpose:** Update an existing vendor
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric vendor ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "string",
  "email": "string",
  "phone": "string",
  "address": "string",
  "logo": "string",
  "status": "string",
  "description": "string",
  "totalPaid": 5000.00,
  "amountPayable": 1200.00
}
```

**Note:** All fields optional for partial update.

**Expected Response 200:**
```json
{
  "message": "Vendor updated successfully",
  "data": { "...full VendorResponse object..." }
}
```

**Frontend Impact:**
- Vendor form submission
- `totalPaid` and `amountPayable` can be manually updated (ledger-style tracking)

---

## DELETE /vendors/:id
**Purpose:** Soft-delete a vendor
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric vendor ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Vendor deleted successfully"
}
```

**Frontend Impact:**
- Vendor removed from list
- Products linked to this vendor retain `vendor_id` reference (backend-dependent cascade behavior)
- Vendor no longer available in product creation dropdowns after deletion
