# Module Overview

**Module Name:** Users (Legacy)
**Internal Identifier:** users
**Purpose:** Legacy user management for the non-SaaS portion of the system. These user accounts are linked to customers, vendors, and staff records as portal login identities. Distinct from `saas_users`.

---

# Business Context

In the legacy system, portal access (for customers, vendors, and staff) is backed by this `users` table. When a customer, vendor, or staff record is created, a corresponding user record is created here with the appropriate role. Role IDs are conventionally `3` for customers, `4` for vendors, and `5` for staff. This module is **not** scoped by `company_id` — all queries are system-wide, making it a global user registry.

This module is distinct from the SaaS user system (`saas_users` table) which manages subscription-level access.

---

# Functional Scope

## Included

- List all users system-wide (no pagination, no company filter)
- Get a single user by ID
- Create a new user with a hashed password
- Update user fields
- Delete a user (soft delete, returns 204)
- Role association via `roles` table

## Excluded

- Company-scoped user listing
- Pagination on the list endpoint
- Authentication / login (this module manages user records, not session tokens)
- SaaS user management (separate table and module)
- Role creation or management through this module

---

# Architecture Notes

- **No `company_id` scoping.** All queries return or operate on users across all companies system-wide. This is intentional legacy behavior.
- All endpoints require a Bearer JWT, but the JWT is not used to filter results by company.
- Passwords are stored as bcrypt hashes. The `password` field is tagged `json:"-"` and is never included in any response.
- The `DELETE` endpoint returns **HTTP 204 No Content** (empty body), unlike most other modules in this system which return HTTP 200 with a message body.
- `POST /users/` and `PUT /users/:id` return the user object **directly** (not wrapped in a `{message, data}` envelope), which is inconsistent with other modules.
- User records preload the associated `Role` on all fetch operations.

---

# Data Model

## Table: `users`

| Column       | Type         | Constraints / Notes                                  |
|--------------|--------------|------------------------------------------------------|
| `id`         | uint         | Primary key, auto-increment                          |
| `username`   | varchar(100) | JSON key: `username`                                 |
| `email`      | varchar(255) | Unique; JSON key: `email`                            |
| `password`   | varchar(255) | bcrypt hashed; JSON tag: `"-"` (never returned)      |
| `role_id`    | uint         | FK → `roles.id`; JSON key: `roleId`                  |
| `address`    | text         | Optional; JSON key: `address`                        |
| `created_at` | timestamp    | JSON key: `createdAt`                                |
| `updated_at` | timestamp    | JSON key: `updatedAt`                                |
| `deleted_at` | timestamp    | Soft delete                                          |

## Table: `roles`

| Column       | Type      | Constraints / Notes           |
|--------------|-----------|-------------------------------|
| `id`         | uint      | Primary key, auto-increment   |
| `title`      | string    | JSON key: `title`             |
| `status`     | bool      | Default: `false`              |
| `created_at` | timestamp | JSON key: `createdAt`         |
| `updated_at` | timestamp | JSON key: `updatedAt`         |
| `deleted_at` | timestamp | Soft delete                   |

## Role ID Conventions

| Role ID | Role       | Used By   |
|---------|------------|-----------|
| 3       | Customer   | Customers module |
| 4       | Vendor     | Vendors module   |
| 5       | Staff      | Staff module     |

## Relationships

| Relationship            | Type       | Notes                                             |
|-------------------------|------------|---------------------------------------------------|
| `User` → `Role`         | Belongs To | Via `role_id`; always preloaded                   |
| `User` ← `Customer`     | Has One    | Via `customer.user_id`; `role_id = 3`             |
| `User` ← `Vendor`       | Has One    | Via `vendor.user_id`; `role_id = 4`               |
| `User` ← `Staff`        | Has One    | Via `staff.user_id`; `role_id = 5`                |

---

# API Endpoints

| Method | Path          | Auth       | Description                                      |
|--------|---------------|------------|--------------------------------------------------|
| GET    | `/users/`     | Bearer JWT | List all users system-wide with roles preloaded  |
| GET    | `/users/:id`  | Bearer JWT | Get a single user with role                      |
| POST   | `/users/`     | Bearer JWT | Create a new user                                |
| PUT    | `/users/:id`  | Bearer JWT | Update user fields (partial)                     |
| DELETE | `/users/:id`  | Bearer JWT | Soft delete a user; returns 204 No Content       |

---

# Request Payloads

## POST `/users/` — Create User

**Content-Type:** `application/json`

```json
{
  "username": "John Doe",
  "email": "john@example.com",
  "password": "plaintextpassword",
  "roleId": 3,
  "address": ""
}
```

| Field      | Notes                                              |
|------------|----------------------------------------------------|
| `username` | Display name                                       |
| `email`    | Must be unique system-wide                         |
| `password` | Sent as plaintext; stored as bcrypt hash           |
| `roleId`   | FK → `roles.id`                                    |
| `address`  | Optional                                           |

## PUT `/users/:id` — Update User

**Content-Type:** `application/json`

Any subset of user fields: `username`, `email`, `password`, `roleId`, `address`.

---

# Response Contracts

## GET `/users/` — 200 OK

```json
{
  "message": "Users retrieved successfully",
  "data": [
    {
      "id": 1,
      "username": "John Doe",
      "email": "john@example.com",
      "roleId": 3,
      "role": {"id": 3, "title": "Customer", "status": true},
      "address": "",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

Note: `password` is never included in responses.

## GET `/users/:id` — 200 OK

```json
{
  "message": "User fetched successfully",
  "data": {
    "id": 1,
    "username": "John Doe",
    "email": "john@example.com",
    "roleId": 3,
    "role": {"id": 3, "title": "Customer", "status": true},
    "address": "",
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**404:** `{"error": "User not found"}`

## POST `/users/` — 201 Created

**Note: Response is a direct user object with no `{message, data}` wrapper.**

```json
{
  "id": 1,
  "username": "John Doe",
  "email": "john@example.com",
  "roleId": 3,
  "address": "",
  "createdAt": "...",
  "updatedAt": "..."
}
```

## PUT `/users/:id` — 200 OK

**Note: Response is a direct user object with no `{message, data}` wrapper.**

```json
{
  "id": 1,
  "username": "John Doe",
  "email": "updated@example.com",
  "roleId": 3,
  "address": "",
  "createdAt": "...",
  "updatedAt": "..."
}
```

## DELETE `/users/:id` — 204 No Content

Empty body. No JSON returned.

---

# Validation Rules

| Field    | Rule                                              |
|----------|---------------------------------------------------|
| `email`  | Must be unique system-wide                        |
| `roleId` | Must reference an existing `roles.id`             |

No other field-level validation rules are documented in the source.

---

# Business Logic Details

### Password Hashing
Passwords are submitted as plaintext and stored as bcrypt hashes. The `password` field carries `json:"-"` and is excluded from all serialized responses.

### No Company Scoping
Unlike all other modules, this module does not filter by `company_id`. All users across all companies are returned and accessible. This is legacy behavior.

### Role Conventions
Role IDs are used by other modules to determine the type of the linked user account:
- `role_id = 3` → Customer
- `role_id = 4` → Vendor
- `role_id = 5` → Staff

These conventions are not enforced by the Users module itself; they are consumed by the Customers, Vendors, and Staff modules respectively.

### DELETE Response Difference
This module returns **HTTP 204 No Content** on delete, with an empty body. All other modules in this system return HTTP 200 with a JSON body. Clients must handle this difference.

### Response Envelope Difference
`POST /users/` and `PUT /users/:id` return the user object directly, without the standard `{"message": "...", "data": {...}}` wrapper used by all other modules. Clients must handle this difference.

---

# Dependencies

| Module      | Reason                                                                   |
|-------------|--------------------------------------------------------------------------|
| `customers` | Customer records link to `users.id` via `customer.user_id` (`role_id=3`) |
| `vendors`   | Vendor records link to `users.id` via `vendor.user_id` (`role_id=4`)     |
| `staff`     | Staff records link to `users.id` via `staff.user_id` (`role_id=5`)       |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- The module is **not** company-scoped. Any authenticated user can list, create, update, or delete any user in the system. This is a known characteristic of the legacy design.
- The `password` field is excluded from all responses via `json:"-"`.
- Passwords are stored as bcrypt hashes; plaintext passwords are never persisted.

---

# Error Handling

| HTTP Status | Trigger                                          |
|-------------|--------------------------------------------------|
| 404         | User not found by ID                             |
| 204         | Successful delete (not an error, but empty body) |

No other specific error status codes are documented for this module.

---

# Testing Notes

- Verify `password` is never returned in any response (GET, POST, PUT).
- Verify password is stored as a bcrypt hash, not plaintext.
- Verify `DELETE /users/:id` returns exactly HTTP 204 with an empty body.
- Verify `POST /users/` returns the user object directly (no `message`/`data` wrapper).
- Verify `PUT /users/:id` returns the user object directly (no `message`/`data` wrapper).
- Verify `GET /users/` returns all users regardless of company; no company filter is applied.
- Verify `email` uniqueness is enforced system-wide (not per company).
- Verify `role` is always preloaded on all fetch operations.

---

# Open Questions / Missing Details

- No documented validation rules beyond `email` uniqueness and `roleId` reference. Whether `username`, `email`, or `password` have required/length constraints is not specified.
- No documented error responses for POST or PUT (e.g., duplicate email, invalid `roleId`).
- Whether the `roles` table is seeded or managed through an admin interface is not documented.
- `roles.status` default is `false`; what this flag controls is not explained.
- Whether the legacy Users module is being deprecated in favor of the SaaS users system is not specified.
- Whether authenticated staff from the SaaS system can create users through this module without restriction is not addressed.

---

# Change Log / Notes

- Source documentation does not include a version history.
- This file was standardized from the original module notes on 2026-03-31.
