# Module Overview

**Module Name:** Customers
**Module Path:** `inventory_management/backend/customers`

This module manages customer records for a company. Each customer is automatically linked to a `users` record (role_id=3) for potential portal access. It supports filtering by status (active/inactive) and customer type (retail/wholesale).

---

# Business Context

Customers represent the end-buyers for a company's products. Each customer record is paired with a system user account to enable future portal login functionality. Customer records are scoped per company and are referenced by orders, shipping addresses, returns, and coupon usage records.

---

# Functional Scope

## Included

- Create, read, update, and delete customer records per company
- Automatic linked `users` record creation on customer create (with default password `"changeme"`)
- Synchronized update of linked user's `username` and `email` when customer name or email changes
- Cascaded soft delete of linked user on customer delete
- Filtering by `status` (active/inactive) and `customerType` (retail/wholesale)
- Paginated list with search across name, email, and phone
- All data scoped by `company_id` extracted from JWT

## Excluded

- Customer portal authentication (the linked user account is provisioned but portal login flow is not documented here)
- Store credit adjustment endpoints (field exists but no dedicated adjustment endpoint is documented)
- Shipping address management (separate module)
- Order/return history retrieval (separate modules)
- Coupon usage tracking (separate module)

---

# Architecture Notes

- All customer records are scoped to a company via `company_id` injected by JWT middleware.
- On create, a corresponding `users` record is inserted first. The resulting user ID is then stored in `customers.user_id`. This is a two-step operation; failure to create the user will return `500` before the customer record is created.
- On update, if `name` or `email` fields are changed, the linked user's `username` and `email` are updated accordingly in the same operation.
- On delete, both the customer and its linked user are soft-deleted.
- `email` is unique per company via a composite index (`idx_customer_company_email`).
- The default password `"changeme"` is bcrypt-hashed before storage.

---

# Data Model

**Table:** `customers`

| Column          | Type         | Constraints / Notes                                               |
|-----------------|--------------|-------------------------------------------------------------------|
| `id`            | uint         | Primary key, auto-increment                                       |
| `company_id`    | uint         | FK → `companies.id`; indexed                                      |
| `user_id`       | *uint        | FK → `users.id`; nullable; linked user account                    |
| `name`          | varchar(255) | Not null                                                          |
| `email`         | varchar(255) | Unique per company: composite index `idx_customer_company_email`  |
| `phone`         | varchar(50)  |                                                                   |
| `address`       | string       |                                                                   |
| `city`          | string       |                                                                   |
| `state`         | string       |                                                                   |
| `zip_code`      | string       | JSON key: `zipCode`                                               |
| `country`       | string       |                                                                   |
| `customer_type` | enum         | `retail` / `wholesale`; default `retail`; JSON key: `customerType` |
| `status`        | enum         | `active` / `inactive`; default `active`                           |
| `notes`         | text         |                                                                   |
| `store_credit`  | float64      | Default `0`; JSON key: `storeCredit`                              |
| `created_at`    | timestamp    | JSON key: `createdAt`                                             |
| `updated_at`    | timestamp    | JSON key: `updatedAt`                                             |
| `deleted_at`    | timestamp    | Soft delete; excluded from JSON responses                         |

**Relationships:**

| Relationship                           | Type       | Notes                                                         |
|----------------------------------------|------------|---------------------------------------------------------------|
| Customer → User                        | BelongsTo  | Via `user_id`; optional; user preloaded on responses          |
| Customer → User.Role                   | Preloaded  | Role preloaded via the linked user                            |
| Customer ← Orders (Sells)              | HasMany    | `sells.customer_id` references this table                     |
| Customer ← ShippingAddresses           | HasMany    | `shipping_addresses.customer_id`                              |
| Customer ← CustomerReturns             | HasMany    | `customer_returns.customer_id`                                |
| Customer ← CouponUsages               | HasMany    | `coupon_usages.customer_id`                                   |

---

# API Endpoints

| Method | Path              | Auth       | Description                                      |
|--------|-------------------|------------|--------------------------------------------------|
| GET    | `/customers/`     | Bearer JWT | Paginated list with search and status/type filter |
| GET    | `/customers/:id`  | Bearer JWT | Single customer with linked user and role        |
| POST   | `/customers/`     | Bearer JWT | Create a customer and linked user account        |
| PUT    | `/customers/:id`  | Bearer JWT | Partial update; syncs linked user if needed      |
| DELETE | `/customers/:id`  | Bearer JWT | Soft delete customer and linked user             |

---

# Request Payloads

### GET `/customers/` — Query Parameters

| Parameter | Type   | Default | Notes                                               |
|-----------|--------|---------|-----------------------------------------------------|
| `page`    | int    | `1`     |                                                     |
| `limit`   | int    | `10`    | Maximum: `100`                                      |
| `search`  | string | —       | LIKE match on `name`, `email`, `phone`              |
| `status`  | string | —       | `active` / `inactive`; `"all"` disables the filter  |
| `type`    | string | —       | `retail` / `wholesale`; `"all"` disables the filter |

### POST `/customers/` — Create Customer

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

| Field          | Type    | Required | Notes                            |
|----------------|---------|----------|----------------------------------|
| `name`         | string  | Yes      |                                  |
| `email`        | string  | Yes      |                                  |
| `phone`        | string  | No       |                                  |
| `address`      | string  | No       |                                  |
| `city`         | string  | No       |                                  |
| `state`        | string  | No       |                                  |
| `zipCode`      | string  | No       |                                  |
| `country`      | string  | No       |                                  |
| `customerType` | string  | No       | `retail` or `wholesale`          |
| `status`       | string  | No       | `active` or `inactive`           |
| `notes`        | string  | No       |                                  |
| `storeCredit`  | float64 | No       | Defaults to `0`                  |

### PUT `/customers/:id` — Update Customer

Accepts any subset of customer fields as a JSON object (partial update via map). Fields not supplied are left unchanged.

---

# Response Contracts

### GET `/customers/` — `200 OK`

```json
{
  "message": "Customers retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "userId": 5,
      "user": {
        "id": 5,
        "username": "John Doe",
        "email": "john@example.com",
        "role": { }
      },
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
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 50,
  "page": 1,
  "limit": 10
}
```

### GET `/customers/:id` — `200 OK`

```json
{
  "message": "Customer fetched successfully",
  "data": { }
}
```

Returns customer object with linked `user` and `role` preloaded.

### POST `/customers/` — `201 Created`

```json
{
  "message": "Customer created successfully",
  "data": { }
}
```

### PUT `/customers/:id` — `200 OK`

```json
{
  "message": "Customer updated successfully",
  "data": { }
}
```

### DELETE `/customers/:id` — `200 OK`

```json
{
  "message": "Customer deleted successfully"
}
```

---

# Validation Rules

| Rule                                  | HTTP Response |
|---------------------------------------|---------------|
| `name` is required                    | `400`         |
| `email` is required                   | `400`         |
| `email` must be unique per company    | (not explicitly documented; see Open Questions) |
| `customerType` must be `retail` or `wholesale` | Not explicitly documented |
| `status` must be `active` or `inactive`        | Not explicitly documented |

---

# Business Logic Details

- **Create flow:**
  1. A `users` record is inserted: `username = name`, `email = email`, `password = bcrypt("changeme")`, `role_id = 3`.
  2. The new user's ID is assigned to `customer.user_id`.
  3. The customer record is saved.
  - If step 1 fails, `500` is returned and no customer record is created.

- **Update flow:** If `name` or `email` is included in the update payload, the linked `users` record is also updated with the new `username` and/or `email`.

- **Delete flow:** The customer record is soft-deleted (`deleted_at` set). The linked user record is also soft-deleted.

- `company_id` is always injected from JWT middleware; it is never accepted from the client.

- The default password `"changeme"` is hashed with bcrypt before storage. The customer portal is expected to prompt users to change this password on first login (see Open Questions).

---

# Dependencies

| Module / Table       | Direction    | Notes                                                           |
|----------------------|--------------|-----------------------------------------------------------------|
| `companies`          | FK (inbound) | `company_id` scopes all customer records                        |
| `users`              | Creates      | A linked user record is created on customer creation            |
| `roles`              | Referenced   | Linked user's role (role_id=3) is preloaded in responses        |
| `sells` (orders)     | Referenced   | Sells reference `customer_id`                                   |
| `shipping_addresses` | Referenced   | Shipping addresses belong to customers                          |
| `customer_returns`   | Referenced   | Returns reference `customer_id`                                 |
| `coupon_usages`      | Referenced   | Coupon usage records reference `customer_id`                    |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is extracted from the JWT and used to scope all queries; users cannot access customers belonging to other companies.
- The default password `"changeme"` is a known weak credential. Customer portal access relies on customers changing this password after first login.
- No role-based access control is documented at the customer module level (see Open Questions).

---

# Error Handling

| HTTP Status | Scenario                                                      |
|-------------|---------------------------------------------------------------|
| `400`       | Missing required fields (`name` or `email`)                   |
| `401`       | Missing or invalid JWT token                                  |
| `404`       | Customer not found for the given `:id`                        |
| `500`       | Failed to create linked user; failed to create/update customer |

**Error response examples:**

```json
{ "error": "Name and email are required" }
{ "error": "Customer not found" }
{ "error": "Failed to create user" }
{ "error": "Failed to create customer" }
```

---

# Testing Notes

- Test that creating a customer also creates a linked `users` record with role_id=3 and hashed `"changeme"` password.
- Test that updating a customer's name or email also updates the linked user.
- Test that deleting a customer also soft-deletes the linked user.
- Test search across `name`, `email`, and `phone` fields.
- Test `status` filter: `active`, `inactive`, and `"all"`.
- Test `type` filter: `retail`, `wholesale`, and `"all"`.
- Test email uniqueness per company: same email in a different company should succeed.
- Test `limit` maximum is enforced at `100`.
- Test that `company_id` from JWT is always applied and cannot be overridden.

---

# Open Questions / Missing Details

- **Email uniqueness error:** The data model documents a unique composite index on `(company_id, email)`, but no explicit `409` error response is documented for duplicate email on create or update. Confirm error behavior.
- **Default password prompt:** There is no documented mechanism to force customers to change the `"changeme"` default password on first portal login. This is a security gap that should be addressed.
- **`role_id = 3` meaning:** The documentation states the linked user is created with `role_id=3` but does not describe what role this corresponds to. This should be documented.
- **Store credit adjustment:** The `store_credit` field exists on the customer record, but there is no dedicated endpoint for crediting or debiting store credit. It is unclear how this value is modified.
- **PUT response:** The documentation does not specify which fields are returned after a partial update. Confirm whether the full customer object (with linked user) is returned.
- **Role-based access control:** No documentation on whether specific SaaS roles have differing permissions on customer endpoints.

---

# Change Log / Notes

- Initial documentation authored from source specification.
- Default password `"changeme"` is a known security concern; flagged for review.
