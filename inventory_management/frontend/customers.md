# Frontend Module: Customers

## Pages
- `/dashboard/customers` — Customer list with search and management
- `/dashboard/customers/new` — Create customer form
- `/dashboard/customers/:id` — Customer detail / edit form

## API Service
`lib/customerApi.ts`

---

## GET /customers
**Purpose:** Fetch paginated list of customers
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by name, email, or phone (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "userId": 10,
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+1234567890",
      "address": "123 Main St",
      "city": "New York",
      "state": "NY",
      "zipCode": "10001",
      "country": "US",
      "customerType": "retail | wholesale",
      "status": "active | inactive",
      "notes": "VIP customer",
      "storeCredit": 50.00,
      "totalOrders": 15,
      "totalSpent": 1250.00,
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-15T00:00:00Z"
    }
  ],
  "total": 120,
  "page": 1,
  "limit": 20
}
```

**Note:** Pagination fields `total`, `page`, `limit` are at top level of response (not nested). `totalOrders` and `totalSpent` are optional computed fields that may not always be present.

**Frontend Impact:**
- Customer list table with search and type/status filters
- `totalSpent` shown as a customer value indicator
- `storeCredit` balance shown per customer
- `customerType` badge: retail / wholesale

---

## GET /customers/:id
**Purpose:** Fetch a single customer by ID
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric customer ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "userId": 10,
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
    "notes": "string",
    "storeCredit": 50.00,
    "totalOrders": 15,
    "totalSpent": 1250.00,
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-15T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Customer detail/edit form pre-populated
- Customer profile shows order history, return history, store credit balance

---

## POST /customers
**Purpose:** Create a new customer
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

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
  "notes": "string",
  "storeCredit": 0.00
}
```

**Note:** `name` and `email` are required. All other fields optional. `customerType` defaults to `"retail"` if omitted. `storeCredit` defaults to `0` if omitted.

**Expected Response 201:**
```json
{
  "message": "Customer created successfully",
  "data": { "...full CustomerResponse object..." }
}
```

**Frontend Impact:**
- New customer appears in customer list and in order creation customer dropdown
- Customer can immediately be selected when creating a new order

---

## PUT /customers/:id
**Purpose:** Update a customer's details
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric customer ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "name": "string",
  "email": "string",
  "phone": "string",
  "address": "string",
  "city": "string",
  "state": "string",
  "zipCode": "string",
  "country": "string",
  "customerType": "retail | wholesale",
  "status": "active | inactive",
  "notes": "string",
  "storeCredit": 75.00
}
```

**Note:** All fields optional for partial update. `storeCredit` can be directly set here (credit top-up or adjustment).

**Expected Response 200:**
```json
{
  "message": "Customer updated successfully",
  "data": { "...full CustomerResponse object..." }
}
```

**Frontend Impact:**
- Customer edit form submission
- Setting `status: "inactive"` prevents customer from being selected in new orders (client-side filter recommended)
- `storeCredit` updates show immediately in customer profile

---

## DELETE /customers/:id
**Purpose:** Delete a customer
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric customer ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Customer deleted successfully"
}
```

**Frontend Impact:**
- Customer removed from list and from order creation dropdown
- Existing orders retain `customerName` as a string (not relational — orders are not deleted)
- Customer returns linked to this customer retain their `customerId` reference (backend cascade behavior TBD)
- Should not allow deletion of customers with active orders (client-side guard recommended)
