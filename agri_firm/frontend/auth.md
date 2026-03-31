# Authentication Module

**Routes:** `/signin`, `/signup`
**Base Path:** `/auth`
**Auth Required:** No (all auth endpoints are public, except `/auth/me` and `/auth/signout`)

---

## TypeScript Interfaces

```typescript
interface AuthUser {
  id: string;
  name: string;
  email: string;
  farmName: string;
  role: "admin" | "manager" | "worker";
}

interface AuthResponse {
  token: string;
  user: AuthUser;
}

interface SignInPayload {
  email: string;
  password: string;
}

interface SignUpPayload {
  name: string;
  email: string;
  password: string;
  confirmPassword: string;
  farmName: string;
  phone: string;
}

interface ForgotPasswordPayload {
  email: string;
}

interface ResetPasswordPayload {
  token: string;
  newPassword: string;
  confirmPassword: string;
}
```

---

## Frontend Storage After Login

After a successful signin or signup, the frontend stores the following in `localStorage`:

| Key              | Value           | Description                         |
|------------------|-----------------|-------------------------------------|
| `token`          | JWT string      | Bearer token for all API calls       |
| `farm_id`        | UUID string     | Active farm ID for multi-tenancy     |
| `user_role`      | string          | Role used for UI permission gating   |

On signout or `401` response from any endpoint, all three keys are cleared and the user is redirected to `/signin`.

---

## Endpoints

---

### POST `/auth/signin`

Sign in with email and password.

**Auth Required:** No

**Request Body:**

```json
{
  "email": "farmer@example.com",
  "password": "SecurePass123!"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Login successful",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "John Doe",
      "email": "farmer@example.com",
      "farmName": "Green Valley Farm",
      "role": "admin"
    }
  }
}
```

**Error Responses:**

| Code  | Scenario                          |
|-------|-----------------------------------|
| `400` | Missing email or password         |
| `401` | Wrong email/password combination  |
| `422` | Email format invalid              |

**Validation Rules:**
- `email`: required, valid email format
- `password`: required, minimum 6 characters

**Frontend Impact:** Failure to return `token` and `user.id` (used as `farm_id` seed or separate `farm_id` field) will break the entire session initialization. The frontend immediately reads `data.token`, `data.user` from this response.

---

### POST `/auth/signup`

Register a new user and create a farm.

**Auth Required:** No

**Request Body:**

```json
{
  "name": "John Doe",
  "email": "farmer@example.com",
  "password": "SecurePass123!",
  "confirmPassword": "SecurePass123!",
  "farmName": "Green Valley Farm",
  "phone": "+1234567890"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Account created successfully",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "John Doe",
      "email": "farmer@example.com",
      "farmName": "Green Valley Farm",
      "role": "admin"
    }
  }
}
```

**Error Responses:**

| Code  | Scenario                              |
|-------|---------------------------------------|
| `400` | Validation errors (see errors array)  |
| `409` | Email already registered              |

**Validation Rules:**
- `name`: required, 2–100 characters
- `email`: required, valid email, must be unique
- `password`: required, minimum 8 characters, at least one number
- `confirmPassword`: required, must match `password`
- `farmName`: required, 2–100 characters
- `phone`: required, valid phone format

**Frontend Impact:** Same as signin — response must include `token` and `user` object. The frontend does not make a separate login call after signup; it uses the token returned here directly.

---

### POST `/auth/signout`

Invalidate the current session server-side (e.g. token blacklist or session removal).

**Auth Required:** Yes (Bearer token)

**Request Body:** None

**Success Response — `200 OK`:**

```json
{
  "message": "Signed out successfully",
  "data": {}
}
```

**Frontend Behavior:** The frontend clears localStorage regardless of the response status. This endpoint is a best-effort server-side invalidation.

---

### GET `/auth/me`

Returns the currently authenticated user's profile and farm information.

**Auth Required:** Yes (Bearer token)

**Query Params:** None

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "farmer@example.com",
    "farmName": "Green Valley Farm",
    "role": "admin",
    "farm_id": "farm-uuid-here",
    "phone": "+1234567890"
  }
}
```

**Error Responses:**

| Code  | Scenario              |
|-------|-----------------------|
| `401` | Token missing/expired |

**Frontend Impact:** This endpoint is called on app initialization to validate the stored token and restore session state. If it returns `401`, the user is logged out. If the response shape changes (e.g. `farm_id` is renamed), the session restoration logic breaks.

---

### POST `/auth/forgot-password`

Request a password reset email.

**Auth Required:** No

**Request Body:**

```json
{
  "email": "farmer@example.com"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Password reset email sent. Please check your inbox.",
  "data": {}
}
```

**Note:** Return `200 OK` even if the email does not exist in the database (to prevent email enumeration attacks). The frontend simply displays the `message` string.

**Validation Rules:**
- `email`: required, valid email format

---

### POST `/auth/reset-password`

Set a new password using the token from the reset email.

**Auth Required:** No

**Request Body:**

```json
{
  "token": "reset-token-from-email-link",
  "newPassword": "NewSecurePass456!",
  "confirmPassword": "NewSecurePass456!"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Password reset successfully. Please sign in.",
  "data": {
    "success": true
  }
}
```

**Error Responses:**

| Code  | Scenario                              |
|-------|---------------------------------------|
| `400` | Passwords do not match, weak password |
| `422` | Token is invalid or expired           |

**Validation Rules:**
- `token`: required
- `newPassword`: required, minimum 8 characters
- `confirmPassword`: required, must match `newPassword`

**Frontend Impact:** On success, the frontend redirects to `/signin`. On `422`, it shows a "link expired" message and prompts the user to request a new reset link.

---

## Notes

- The JWT token's expiry should be reasonable for a farm management app (e.g. 7 days or 30 days). Short expiry (1 hour) with refresh tokens is ideal but adds complexity — document whichever strategy is chosen.
- If refresh tokens are implemented, a `POST /auth/refresh` endpoint must be added and the frontend interceptor must be updated.
- The `farm_id` returned in `/auth/me` or embedded in the JWT payload must match what is stored in `localStorage.farm_id` for the multi-tenancy system to work correctly.
