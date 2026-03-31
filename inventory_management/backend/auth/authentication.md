# Module Overview

**Module Name:** Authentication
**Module Path:** `inventory_management/backend/auth`

This module handles user authentication for two distinct systems: a legacy user system and a SaaS multi-tenant user system. It provides login, logout, JWT token issuance, token blacklisting, SaaS signup with email verification, password reset via email, password update, and current-user retrieval.

---

# Business Context

The platform operates two parallel authentication systems:

1. **Legacy system** — Uses the `users` table. Intended for internal or non-SaaS use. Tokens carry only `user_id`.
2. **SaaS system** — Uses the `saas_users` table. Supports multi-tenancy via `company_id`. Tokens carry `user_id`, `company_id`, and `email`. All business data queries are scoped by `company_id` derived from the token.

SaaS signup creates a company and an owner user, but the account remains `"unverified"` until the user clicks the verification link sent to their email. Only after email verification does the trial subscription and company settings get activated and a JWT issued.

Forgot password sends a reset link to the user's email. The reset link is never exposed in the API response. After resetting, the user must log in again.

Authenticated users can update their own password at any time via the `POST /auth/update-password` endpoint, which requires the current password to be confirmed before setting a new one.

---

# Functional Scope

## Included

- Legacy login and logout (`/login`, `/logout`)
- SaaS signup with email verification (`/auth/signup` → `/auth/verify-email`)
- Resend verification email (`/auth/resend-verification`)
- SaaS login — blocked if account is unverified (`/auth/login`)
- Password reset via email link (`/auth/forgot-password`, `/auth/reset-password`)
- Update password for authenticated user (`/auth/update-password`)
- Current user and company info retrieval (`/auth/me`)
- SaaS logout (`/auth/logout`)
- JWT issuance (HS256, 24-hour expiry)
- In-memory token blacklisting on logout
- SMTP-based email delivery for verification and password reset

## Excluded

- OAuth or third-party SSO
- Multi-factor authentication
- Session-based authentication
- Role permission management (handled by staff-roles module)

---

# Architecture Notes

- Two JWT structures exist:
  - **Legacy JWT:** contains only `user_id`
  - **SaaS JWT:** contains `user_id`, `company_id`, and `email`
- JWT algorithm: HS256. Secret: `JWT_SECRET` environment variable. Expiry: 24 hours.
- Token blacklist is stored **in-memory**; it resets on server restart.
- JWT middleware validates the token, checks the blacklist, and injects `user_id`, `company_id`, and `email` into the request context.
- All protected routes require `Authorization: Bearer <token>`.
- `company_id` from the JWT is the tenant isolation key for all downstream queries.
- The `password` field on both `users` and `saas_users` is tagged `json:"-"` and is never included in any API response.
- The owner role cannot be changed or removed.
- **Email is sent** via SMTP using environment variables. The system sends two types of email:
  1. Email verification link on signup (and on resend request)
  2. Password reset link on forgot-password request
- The `resetLink` is **never returned in the API response** — it is sent to the user's inbox only.
- Email verification tokens have a **24-hour TTL**. Password reset tokens have a **1-hour TTL**.
- A new user account stays in `status = "unverified"` until email is verified. Unverified accounts cannot log in.

### SMTP Environment Variables

| Variable          | Example                      | Description                     |
|-------------------|------------------------------|---------------------------------|
| `SMTP_HOST`       | `smtp.sendgrid.net`          | SMTP server host                |
| `SMTP_PORT`       | `587`                        | SMTP port (TLS)                 |
| `SMTP_USERNAME`   | `apikey`                     | SMTP username                   |
| `SMTP_PASSWORD`   | `SG.xxxx`                    | SMTP password or API key        |
| `SMTP_FROM`       | `noreply@yourdomain.com`     | Sender email address            |
| `SMTP_FROM_NAME`  | `YourApp`                    | Sender display name             |
| `FRONTEND_URL`    | `https://app.yourdomain.com` | Used to build verification and reset links in emails |

---

# Data Model

### Table: `users` (Legacy)

| Column       | Type         | Constraints / Notes                        |
|--------------|--------------|--------------------------------------------|
| `id`         | uint         | Primary key, auto-increment                |
| `username`   | varchar(100) |                                            |
| `email`      | varchar(255) | Unique                                     |
| `password`   | varchar(255) | bcrypt hashed; never returned in responses |
| `role_id`    | uint         | FK → `roles.id`                            |
| `address`    | text         | Optional                                   |
| `created_at` | timestamp    | Auto-managed                               |
| `updated_at` | timestamp    | Auto-managed                               |
| `deleted_at` | timestamp    | Soft delete                                |

### Table: `roles` (Legacy)

| Column       | Type      | Constraints / Notes |
|--------------|-----------|---------------------|
| `id`         | uint      | Primary key         |
| `title`      | string    |                     |
| `status`     | bool      | Default `false`     |
| `created_at` | timestamp |                     |
| `updated_at` | timestamp |                     |
| `deleted_at` | timestamp | Soft delete         |

### Table: `saas_users` (SaaS)

| Column        | Type         | Constraints / Notes                                          |
|---------------|--------------|--------------------------------------------------------------|
| `id`          | uint         | Primary key                                                  |
| `company_id`  | uint         | FK → `companies.id`                                          |
| `email`       | varchar(255) | Unique per company; composite index `idx_company_email`      |
| `full_name`   | string       |                                                              |
| `password`    | string       | bcrypt hashed; `json:"-"`, never returned in responses       |
| `role`        | string       | `owner` / `admin` / `manager` / `staff`                      |
| `role_id`     | *uint        | FK → `staff_roles.id`; optional                              |
| `status`      | string       | `unverified` / `active` / `invited` / `inactive`; default `unverified` on signup |
| `joined_date` | timestamp    |                                                              |
| `last_login`  | *timestamp   | Nullable                                                     |
| `avatar`      | string       | Optional                                                     |
| `uploaded_by` | *uint        | FK → `saas_users.id`; nullable; who uploaded the avatar      |
| `created_at`  | timestamp    |                                                              |
| `updated_at`  | timestamp    |                                                              |
| `deleted_at`  | timestamp    | Soft delete                                                  |

> **Note:** `status = "unverified"` is the new initial state for all new SaaS signups. Previously the default was `"active"`. Users with `status = "unverified"` cannot log in.

### Table: `email_verifications` (New)

| Column       | Type         | Constraints / Notes                                              |
|--------------|--------------|------------------------------------------------------------------|
| `id`         | uint         | Primary key, auto-increment                                      |
| `user_id`    | uint         | FK → `saas_users.id`                                             |
| `email`      | varchar(255) | Indexed                                                          |
| `token`      | varchar(255) | Unique; random 32-byte hex token                                 |
| `expires_at` | timestamp    | 24-hour TTL from creation                                        |
| `status`     | string       | `pending` / `used` / `expired`                                   |
| `used_at`    | *timestamp   | Nullable; set when token is consumed                             |
| `created_at` | timestamp    |                                                                  |
| `updated_at` | timestamp    |                                                                  |

### Table: `password_resets`

| Column        | Type         | Constraints / Notes                  |
|---------------|--------------|--------------------------------------|
| `id`          | uint         | Primary key                          |
| `user_id`     | uint         | FK → `saas_users.id`                 |
| `email`       | varchar(255) | Indexed                              |
| `reset_token` | varchar(255) | Unique index                         |
| `expires_at`  | timestamp    | 1-hour TTL from creation             |
| `status`      | string       | `pending` / `used` / `expired`       |
| `used_at`     | *timestamp   | Nullable; set when token is consumed |
| `created_at`  | timestamp    |                                      |
| `updated_at`  | timestamp    |                                      |

**Relationships:**

| Relationship                                  | Type      | Notes                               |
|-----------------------------------------------|-----------|-------------------------------------|
| `users.role_id` → `roles.id`                  | BelongsTo |                                     |
| `saas_users.company_id` → `companies.id`      | BelongsTo | Tenant scoping                      |
| `saas_users.role_id` → `staff_roles.id`       | BelongsTo | Optional; granular permissions      |
| `email_verifications.user_id` → `saas_users.id` | BelongsTo |                                   |
| `password_resets.user_id` → `saas_users.id`  | BelongsTo |                                     |

---

# API Endpoints

| Method | Path                          | Auth       | System | Description                                           |
|--------|-------------------------------|------------|--------|-------------------------------------------------------|
| POST   | `/login`                      | None       | Legacy | Legacy login; returns user-scoped JWT                 |
| POST   | `/logout`                     | Bearer JWT | Legacy | Blacklists token in-memory                            |
| POST   | `/auth/signup`                | None       | SaaS   | Creates company + owner user; sends verification email|
| POST   | `/auth/verify-email`          | None       | SaaS   | Verifies email token; activates account; returns JWT  |
| POST   | `/auth/resend-verification`   | None       | SaaS   | Resends verification email to unverified user         |
| POST   | `/auth/login`                 | None       | SaaS   | SaaS login; blocked if unverified                     |
| POST   | `/auth/forgot-password`       | None       | SaaS   | Sends password reset link to email                    |
| POST   | `/auth/reset-password`        | None       | SaaS   | Resets password using a valid reset token             |
| POST   | `/auth/update-password`       | Bearer JWT | SaaS   | Updates password for the authenticated user           |
| GET    | `/auth/me`                    | Bearer JWT | SaaS   | Returns current user and company info                 |
| POST   | `/auth/logout`                | Bearer JWT | SaaS   | SaaS logout                                           |

---

# Request Payloads

### POST `/login` — Legacy Login

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

| Field      | Type   | Required | Notes |
|------------|--------|----------|-------|
| `email`    | string | Yes      |       |
| `password` | string | Yes      |       |

---

### POST `/auth/signup` — SaaS Signup

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

| Field           | Type   | Required | Notes                 |
|-----------------|--------|----------|-----------------------|
| `companyName`   | string | Yes      |                       |
| `ownerFullName` | string | Yes      |                       |
| `email`         | string | Yes      | Must be a valid email |
| `phone`         | string | Yes      |                       |
| `password`      | string | Yes      | Minimum 8 characters  |
| `businessType`  | string | No       |                       |
| `website`       | string | No       |                       |
| `country`       | string | No       |                       |

---

### POST `/auth/verify-email` — Email Verification

```json
{
  "token": "<verification_token>"
}
```

| Field   | Type   | Required | Notes                                    |
|---------|--------|----------|------------------------------------------|
| `token` | string | Yes      | Token from the verification email link   |

---

### POST `/auth/resend-verification` — Resend Verification Email

```json
{
  "email": "john@acme.com"
}
```

| Field   | Type   | Required | Notes |
|---------|--------|----------|-------|
| `email` | string | Yes      |       |

---

### POST `/auth/login` — SaaS Login

```json
{
  "email": "john@acme.com",
  "password": "securepassword"
}
```

| Field      | Type   | Required | Notes                 |
|------------|--------|----------|-----------------------|
| `email`    | string | Yes      | Must be a valid email |
| `password` | string | Yes      |                       |

---

### POST `/auth/forgot-password` — Forgot Password

```json
{
  "email": "john@acme.com"
}
```

| Field   | Type   | Required | Notes |
|---------|--------|----------|-------|
| `email` | string | Yes      |       |

---

### POST `/auth/reset-password` — Reset Password

```json
{
  "token": "<reset_token>",
  "newPassword": "newpassword123",
  "confirmPassword": "newpassword123"
}
```

| Field             | Type   | Required | Notes                        |
|-------------------|--------|----------|------------------------------|
| `token`           | string | Yes      |                              |
| `newPassword`     | string | Yes      | Minimum 8 characters         |
| `confirmPassword` | string | Yes      | Must match `newPassword`     |

---

### POST `/auth/update-password` — Update Password (Authenticated)

```json
{
  "currentPassword": "oldpassword123",
  "newPassword": "newpassword456",
  "confirmPassword": "newpassword456"
}
```

| Field             | Type   | Required | Notes                              |
|-------------------|--------|----------|------------------------------------|
| `currentPassword` | string | Yes      | Must match the user's stored password |
| `newPassword`     | string | Yes      | Minimum 8 characters               |
| `confirmPassword` | string | Yes      | Must match `newPassword`           |

---

# Response Contracts

### POST `/login` — `200 OK` (Legacy)

```json
{
  "message": "Login successful",
  "token": "<jwt>",
  "expires": "2024-01-02T15:04:05Z"
}
```

---

### POST `/logout` — `200 OK` (Legacy)

```json
{
  "message": "Logged out successfully"
}
```

---

### POST `/auth/signup` — `201 Created`

> Account is created but **not yet active**. No JWT is returned. User must verify their email first.

```json
{
  "message": "Account created successfully. Please check your email to verify your account.",
  "data": {
    "userId": 1,
    "companyId": 1,
    "email": "john@acme.com",
    "companyName": "Acme Corp",
    "status": "unverified"
  }
}
```

---

### POST `/auth/verify-email` — `200 OK`

> Email verified. Trial activated. JWT returned — user is now logged in.

```json
{
  "message": "Email verified successfully. Trial activated for 10 days.",
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
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  }
}
```

---

### POST `/auth/resend-verification` — `200 OK`

> Always returns `200` regardless of whether the email exists or is already verified (prevents enumeration).

```json
{
  "message": "If that email is registered and unverified, a new verification link has been sent."
}
```

---

### POST `/auth/login` — `200 OK`

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
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z"
    }
  }
}
```

---

### POST `/auth/forgot-password` — `200 OK`

> Always returns `200` regardless of whether the email exists (prevents enumeration). Reset link is sent to inbox only — never in the response body.

```json
{
  "message": "If that email is registered, a password reset link has been sent."
}
```

---

### POST `/auth/reset-password` — `200 OK`

```json
{
  "message": "Password reset successfully.",
  "data": null
}
```

---

### POST `/auth/update-password` — `200 OK`

```json
{
  "message": "Password updated successfully.",
  "data": null
}
```

---

### GET `/auth/me` — `200 OK`

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
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-01T00:00:00Z",
      "trialDaysRemaining": 8,
      "subscriptionEndDate": "2024-01-11T00:00:00Z"
    }
  }
}
```

---

### POST `/auth/logout` — `200 OK` (SaaS)

```json
{
  "message": "Logged out successfully",
  "data": null
}
```

---

# Validation Rules

| Rule                                                        | Endpoint(s)                         | HTTP Code |
|-------------------------------------------------------------|-------------------------------------|-----------|
| `email` is a valid email format                             | signup, login, forgot-password      | `400`     |
| `companyName` is required                                   | signup                              | `400`     |
| `ownerFullName` is required                                 | signup                              | `400`     |
| `phone` is required                                         | signup                              | `400`     |
| `password` minimum 8 characters                             | signup, reset-password, update-password | `400` |
| `newPassword` must equal `confirmPassword`                  | reset-password, update-password     | `400`     |
| `currentPassword` must match the stored password            | update-password                     | `400`     |
| `token` is required and must not be expired or already used | verify-email, reset-password        | `400`     |
| Email must not already be registered                        | signup                              | `409`     |
| Account must have `status = "active"` to log in            | login                               | `403`     |

---

# Business Logic Details

### Signup Flow (Updated)

1. Validate request payload
2. Check email is not already registered → `409` if duplicate
3. Hash password with bcrypt
4. Create `Company` record (status: `"trial"` — but trial not yet activated)
5. Create `SaasUser` record (status: `"unverified"`, role: `"owner"`)
6. Generate random 32-byte hex email verification token
7. Insert `email_verifications` record (TTL: 24 hours, status: `"pending"`)
8. Send verification email via SMTP containing:
   - Link: `{FRONTEND_URL}/auth/verify-email?token=<token>`
   - Subject: `"Verify your email address"`
9. Return `201` with basic account info — **no JWT yet**

### Email Verification Flow (New)

1. Receive `token` from request
2. Look up `email_verifications` by token where `status = "pending"` and `expires_at > now`
3. If not found or expired → `400`
4. Mark `email_verifications.status = "used"`, set `used_at`
5. Update `saas_users.status = "active"`
6. Create `Subscription` (10-day trial, `plan_id = 1`)
7. Create `CompanySettings` with defaults
8. Update `Company.status = "trial"`
9. Issue JWT (SaaS claims: `user_id`, `company_id`, `email`)
10. Return `200` with full login response including JWT

### Resend Verification Flow (New)

1. Always return `200` (prevent email enumeration)
2. Look up `saas_users` by email where `status = "unverified"`
3. If found: invalidate any existing `pending` verification tokens for this user
4. Generate new token, insert new `email_verifications` record (24-hour TTL)
5. Send verification email

### Login Flow (Updated)

1. Validate credentials
2. Look up user by email
3. If not found or password mismatch → `401`
4. If `status = "unverified"` → `403` with message: `"Please verify your email before logging in."`
5. If `status = "inactive"` → `403`
6. Verify bcrypt password match
7. Update `last_login`
8. Issue JWT
9. Return `200` with full login response

### Forgot Password Flow (Updated)

1. Always return `200` (prevent email enumeration)
2. Look up `saas_users` by email
3. If not found: return `200` silently — no email sent
4. Invalidate any existing `pending` reset tokens for this user
5. Generate new reset token (32-byte hex)
6. Insert `password_resets` record (TTL: 1 hour, status: `"pending"`)
7. Send reset email via SMTP containing:
   - Link: `{FRONTEND_URL}/auth/reset-password?token=<token>`
   - Subject: `"Reset your password"`
8. **Do not return the reset link in the response body**

### Reset Password Flow

1. Validate `token`, `newPassword`, `confirmPassword`
2. Check `newPassword == confirmPassword` → `400` if mismatch
3. Look up `password_resets` by token where `status = "pending"` and `expires_at > now`
4. If not found or expired → `400`
5. Hash new password with bcrypt
6. Update `saas_users.password`
7. Mark `password_resets.status = "used"`, set `used_at`
8. Return `200`

### Update Password Flow (New)

1. Extract `user_id` from JWT context (authenticated endpoint)
2. Validate `currentPassword`, `newPassword`, `confirmPassword`
3. Check `newPassword == confirmPassword` → `400` if mismatch
4. Fetch `saas_users` record by `user_id`
5. bcrypt-compare `currentPassword` against stored hash → `400` if mismatch
6. Ensure `newPassword` is not the same as `currentPassword` → `400`
7. Hash `newPassword` with bcrypt
8. Update `saas_users.password`
9. Return `200`

### JWT Claims

**SaaS token:**
```json
{
  "user_id": 1,
  "company_id": 1,
  "email": "john@acme.com",
  "exp": 1700000000,
  "iat": 1699913600
}
```

**Legacy token:** Contains only `user_id`.

### Other Rules

- Passwords are always bcrypt-hashed before storage. The `password` field is never serialized to JSON responses (`json:"-"`).
- The owner role cannot be changed or removed.
- Token blacklist is in-memory and resets on server restart.

---

# Dependencies

| Module / Table       | Direction    | Notes                                                        |
|----------------------|--------------|--------------------------------------------------------------|
| `companies`          | FK (inbound) | SaaS users and subscriptions are scoped to a company         |
| `roles`              | FK (inbound) | Legacy user roles                                            |
| `staff_roles`        | FK (inbound) | Optional granular role for SaaS users                        |
| `subscriptions`      | Created by   | Trial subscription created on email verification             |
| `company_settings`   | Created by   | Company settings created on email verification               |
| `email_verifications`| Owned by     | Created on signup and resend-verification                    |
| `password_resets`    | Owned by     | Created on forgot-password                                   |
| SMTP service         | External     | Required for sending verification and reset emails           |

---

# Security / Permissions

- All protected endpoints require `Authorization: Bearer <token>`.
- JWT middleware validates token signature, checks in-memory blacklist, and rejects expired tokens.
- `company_id` from the JWT is the tenant isolation key for all downstream queries.
- Passwords are bcrypt-hashed and never exposed in any response.
- The `password` struct field is tagged `json:"-"` to prevent accidental serialization.
- The owner role is protected from modification or removal at the application level.
- The forgot-password and resend-verification endpoints never reveal whether an email exists in the system (always return `200`).
- Reset links and verification links are **sent to email only** — never returned in API responses.
- Email verification tokens expire in **24 hours**. Password reset tokens expire in **1 hour**.
- `update-password` requires the current password to be confirmed, preventing account takeover from an unattended session.
- Unverified accounts (`status = "unverified"`) are blocked from logging in.

---

# Error Handling

| HTTP Status | Scenario                                                                                          |
|-------------|---------------------------------------------------------------------------------------------------|
| `400`       | Invalid/missing fields; passwords do not match; `currentPassword` incorrect; invalid/expired token |
| `401`       | Invalid credentials on login; missing/invalid/expired JWT; token blacklisted                      |
| `403`       | Account is `unverified` — must verify email before logging in; account is `inactive`              |
| `409`       | Email already registered on signup                                                                |
| `500`       | Token generation failure; SMTP send failure; DB errors                                            |

**Common error response formats:**

```json
{ "error": "Invalid email or password" }
{ "message": "Email already registered" }
{ "message": "Please verify your email before logging in." }
{ "message": "Passwords do not match" }
{ "message": "Current password is incorrect" }
{ "message": "New password must be different from the current password" }
{ "message": "Invalid or expired verification token" }
{ "message": "Invalid or expired reset token" }
```

---

# Testing Notes

- **Signup:** Confirm account is created with `status = "unverified"`. Confirm no JWT in response. Confirm verification email is sent (check SMTP mock).
- **Verify email:** Confirm account activates, trial subscription is created, company settings created, JWT returned.
- **Verify with expired token:** Confirm `400` returned.
- **Verify with already-used token:** Confirm `400` returned.
- **Resend verification:** Confirm old token is invalidated. Confirm new email sent. Confirm `200` for unknown email (no leak).
- **Login blocked for unverified:** Confirm `403` with clear message.
- **Login after verification:** Confirm `200` with JWT.
- **Forgot password:** Confirm reset email sent. Confirm `200` for unknown email (no leak). Confirm reset link is NOT in response body.
- **Reset password:** Test valid token, expired token, used token, mismatched passwords.
- **Update password:** Test correct current password, wrong current password, mismatched new passwords, same new as old.
- **JWT claims:** Confirm SaaS JWT has `user_id`, `company_id`, `email`. Confirm legacy JWT has only `user_id`.
- **Token blacklist:** Confirm blacklisted token is rejected. Confirm blacklist clears on server restart.
- **`/auth/me`:** Confirm returns current user and accurate `trialDaysRemaining`.
- **Password never in response:** Confirm `password` field absent from all API responses.

---

# Open Questions / Missing Details

- **SaaS logout blacklisting:** It is unclear whether the SaaS `/auth/logout` also adds the token to the in-memory blacklist server-side (as the legacy `/logout` does) or only instructs the client to discard it. Should be confirmed and documented.
- **Token blacklist in production:** The in-memory blacklist resets on server restart and does not work across multiple instances. For production with rolling restarts or horizontal scaling, a distributed store (e.g., Redis) is required.
- **`plan_id = 1` hardcoded:** Signup hardcodes `plan_id = 1` for the trial subscription. The plan content and any future plan changes should be tracked explicitly.
- **`licenseKey: "trial-1"` format:** The generation logic for this field is not documented.
- **SMTP failure handling:** If the SMTP send fails during signup, should the user record be rolled back or should signup succeed silently and allow a resend? This needs a defined policy.
- **Legacy system future:** It is unclear whether `/login` and `/logout` are intended for long-term use or will be deprecated in favour of the SaaS auth system.
- **Verification email expiry for invited users:** Team invite flow (`/auth/accept-invitation`) creates users differently. It is unclear whether the email verification flow applies to invited users.
- **`update-password` for legacy users:** The `POST /auth/update-password` endpoint is defined for SaaS users only. Legacy users have no equivalent endpoint.

---

# Change Log / Notes

- **Added:** Email verification flow on signup — account starts as `unverified`, JWT only issued after verification.
- **Added:** `email_verifications` table with 24-hour TTL tokens.
- **Added:** `POST /auth/verify-email` endpoint.
- **Added:** `POST /auth/resend-verification` endpoint.
- **Added:** `status = "unverified"` as a new `saas_users` status value.
- **Added:** Login now blocks `unverified` users with `403`.
- **Added:** `POST /auth/update-password` endpoint for authenticated password changes (requires current password confirmation).
- **Updated:** Forgot-password response no longer returns `resetLink` — link is sent to email only.
- **Updated:** Trial subscription and company settings are now provisioned at email verification time, not at signup time.
- **Updated:** SMTP environment variables documented.
- **Fixed:** `resetLink` exposure in forgot-password response — flagged as development shortcut, now removed from contract.
