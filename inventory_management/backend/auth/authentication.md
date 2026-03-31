# Module: Authentication

## Business Purpose
Handles user authentication for both the legacy user system and the SaaS user system. Provides login/logout via JWT tokens and blacklists tokens on logout. The SaaS auth system also handles signup, password reset, and fetching the current logged-in user.

---

## Database Tables

### Table: `users`
Used by the legacy auth system (Login/Logout).

| Column       | Type           | Notes                          |
|--------------|----------------|--------------------------------|
| id           | uint (PK, AI)  | Primary key                    |
| username     | varchar(100)   |                                |
| email        | varchar(255)   | Unique                         |
| password     | varchar(255)   | bcrypt hashed, never returned  |
| role_id      | uint (FK)      | FK → roles.id                  |
| address      | text           | Optional                       |
| created_at   | timestamp      | Auto                           |
| updated_at   | timestamp      | Auto                           |
| deleted_at   | timestamp      | Soft delete                    |

### Table: `roles`
| Column     | Type    | Notes       |
|------------|---------|-------------|
| id         | uint PK |             |
| title      | string  |             |
| status     | bool    | default false |
| created_at | timestamp |           |
| updated_at | timestamp |           |
| deleted_at | timestamp | Soft delete |

### Table: `saas_users`
Used by the SaaS auth system.

| Column      | Type          | Notes                                        |
|-------------|---------------|----------------------------------------------|
| id          | uint (PK)     |                                              |
| company_id  | uint (FK)     | FK → companies.id                            |
| email       | varchar(255)  | Unique per company: idx_company_email        |
| full_name   | string        |                                              |
| password    | string        | bcrypt hashed, json:"-" (never returned)     |
| role        | string        | owner / admin / manager / staff              |
| role_id     | *uint         | FK → staff_roles.id (optional)               |
| status      | string        | active / invited / inactive; default active  |
| joined_date | timestamp     |                                              |
| last_login  | *timestamp    | nullable                                     |
| avatar      | string        | optional                                     |
| created_at  | timestamp     |                                              |
| updated_at  | timestamp     |                                              |

### Table: `password_resets`
| Column      | Type         | Notes                        |
|-------------|--------------|------------------------------|
| id          | uint (PK)    |                              |
| user_id     | uint (FK)    | FK → saas_users.id           |
| email       | varchar(255) | Indexed                      |
| reset_token | varchar(255) | Unique index                 |
| expires_at  | timestamp    | 1 hour TTL                   |
| status      | string       | pending / used / expired     |
| used_at     | *timestamp   | nullable                     |
| created_at  | timestamp    |                              |
| updated_at  | timestamp    |                              |

---

## Relationships
- `users.role_id` → `roles.id`
- `saas_users.company_id` → `companies.id`
- `saas_users.role_id` → `staff_roles.id` (optional)
- `password_resets.user_id` → `saas_users.id`

---

## Endpoints

### POST /login
**Auth:** None
**Purpose:** Legacy user login. Returns a JWT token scoped to user_id only (no company_id in claims).

**Request Body (JSON):**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response 200:**
```json
{
  "message": "Login successful",
  "token": "<jwt>",
  "expires": "2024-01-02T15:04:05Z"
}
```

**Error Responses:**
- `400` — `{"error": "Invalid login data"}`
- `401` — `{"error": "Invalid email or password"}`
- `500` — `{"error": "Failed to generate token"}`

---

### POST /logout
**Auth:** Bearer JWT required
**Purpose:** Blacklists current token in memory (clears on server restart).

**Request Headers:**
```
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "message": "Logged out successfully"
}
```

**Error Responses:**
- `400` — `{"error": "Token missing"}`
- `401` — `{"error": "Authorization header missing"}` / `{"error": "Invalid authorization format"}` / `{"error": "Invalid or expired token"}` / `{"error": "Token has been logged out"}`

---

### POST /auth/signup
**Auth:** None
**Purpose:** SaaS signup. Creates a Company + owner SaasUser + trial Subscription + CompanySettings in one flow.

**Request Body (JSON):**
```json
{
  "companyName": "Acme Corp",
  "ownerFullName": "John Doe",
  "email": "john@acme.com",
  "phone": "+1234567890",
  "password": "securepassword",
  "businessType": "Retail",
  "website": "https://acme.com",
  "country": "US"
}
```

**Validation:**
- `companyName` — required
- `ownerFullName` — required
- `email` — required, valid email
- `phone` — required
- `password` — required, min 8 chars

**Response 201:**
```json
{
  "message": "Company created successfully. Trial activated for 10 days.",
  "data": {
    "userId": 1,
    "companyId": 1,
    "email": "john@acme.com",
    "userEmail": "john@acme.com",
    "companyName": "Acme Corp",
    "userRole": "owner",
    "companyStatus": "trial",
    "token": "<jwt>",
    "licenseKey": "trial-1",
    "licenseType": "trial",
    "trialStartDate": "2024-01-01T00:00:00Z",
    "trialEndDate": "2024-01-11T00:00:00Z",
    "trialDaysRemaining": 10,
    "company": {
      "id": 1,
      "name": "Acme Corp",
      "status": "trial",
      "createdAt": "...",
      "updatedAt": "..."
    }
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid request", "error": "..."}`
- `409` — `{"message": "Email already registered"}`
- `500` — various server errors

---

### POST /auth/login
**Auth:** None
**Purpose:** SaaS user login. Returns JWT with user_id, company_id, email claims.

**Request Body (JSON):**
```json
{
  "email": "john@acme.com",
  "password": "securepassword"
}
```

**Validation:**
- `email` — required, valid email
- `password` — required

**Response 200:**
```json
{
  "message": "Login successful",
  "data": {
    "userId": 1,
    "userEmail": "john@acme.com",
    "companyId": 1,
    "companyName": "Acme Corp",
    "companyStatus": "trial",
    "userRole": "owner",
    "token": "<jwt>",
    "licenseKey": "trial-1",
    "licenseType": "trial",
    "email": "john@acme.com",
    "company": {
      "id": 1,
      "name": "Acme Corp",
      "status": "trial",
      "createdAt": "...",
      "updatedAt": "..."
    }
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid request", "error": "..."}`
- `401` — `{"message": "Invalid email or password"}`
- `500` — `{"message": "Failed to fetch company"}`

---

### POST /auth/forgot-password
**Auth:** None
**Purpose:** Initiates password reset. Creates a PasswordReset record. Always returns 200 (doesn't reveal if email exists).

**Request Body (JSON):**
```json
{
  "email": "john@acme.com"
}
```

**Response 200:**
```json
{
  "message": "Password reset link has been sent to your email",
  "data": {
    "resetLink": "http://frontend/auth/reset-password?token=<token>"
  }
}
```

> **Note:** `resetLink` is returned in response for development/testing. Remove in production.

---

### POST /auth/reset-password
**Auth:** None
**Purpose:** Resets password using a valid token (expires in 1 hour).

**Request Body (JSON):**
```json
{
  "token": "<reset_token>",
  "newPassword": "newpassword123",
  "confirmPassword": "newpassword123"
}
```

**Validation:**
- `token` — required
- `newPassword` — required, min 8 chars
- `confirmPassword` — required, min 8 chars
- `newPassword` must equal `confirmPassword`

**Response 200:**
```json
{
  "message": "Password reset successfully",
  "data": null
}
```

**Error Responses:**
- `400` — `{"message": "Passwords do not match"}` / `{"message": "Invalid or expired reset token"}`

---

### GET /auth/me
**Auth:** Bearer JWT required
**Purpose:** Returns current authenticated SaaS user and company info.

**Response 200:**
```json
{
  "message": "Current user fetched",
  "data": {
    "user": {
      "id": 1,
      "companyId": 1,
      "email": "john@acme.com",
      "fullName": "John Doe",
      "role": "owner",
      "status": "active",
      "joinedDate": "2024-01-01T00:00:00Z",
      "lastLogin": "2024-01-02T10:00:00Z"
    },
    "company": {
      "id": 1,
      "name": "Acme Corp",
      "status": "trial",
      "createdAt": "...",
      "updatedAt": "...",
      "trialDaysRemaining": 8,
      "subscriptionEndDate": "2024-01-11T00:00:00Z"
    }
  }
}
```

---

### POST /auth/logout
**Auth:** Bearer JWT required
**Purpose:** SaaS logout. Returns success (client should discard token).

**Response 200:**
```json
{
  "message": "Logged out successfully",
  "data": null
}
```

---

## JWT Structure

JWT claims (both legacy and SaaS tokens signed with HS256):
```json
{
  "user_id": 1,
  "company_id": 1,
  "email": "john@acme.com",
  "exp": 1700000000,
  "iat": 1699913600
}
```

> **Note:** Legacy `/login` token (via utils.GenerateJWT) only contains `user_id`. SaaS `/auth/login` token contains `user_id`, `company_id`, and `email`.

**Expiry:** 24 hours
**Algorithm:** HS256
**Secret:** `JWT_SECRET` env variable

---

## Authentication & Authorization Rules
- All protected routes require `Authorization: Bearer <token>` header
- Middleware validates token, checks blacklist, then sets `user_id`, `company_id`, `email` in context
- `company_id` is used to scope all business data queries
- Tokens are blacklisted in-memory on logout — resets on server restart
- Owner role cannot be changed or removed

---

## Business Logic Notes
- On SaaS signup: company status = `"trial"`, subscription auto-created with 10-day trial, plan_id = 1
- Password always hashed with bcrypt before storage
- `password` field is never serialized in JSON responses (json:"-")
