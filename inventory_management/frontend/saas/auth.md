# Frontend Module: SaaS Authentication

## Pages
- `/auth/login` — Login
- `/auth/signup` — Signup
- `/auth/forgot-password` — Forgot password
- `/auth/reset-password` — Reset password with token

## API Service
`lib/saasAuthApi.ts`

---

## POST /auth/signup
**Purpose:** Create new company + owner account with 10-day trial.
**Auth:** None
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "companyName": "string (required)",
  "ownerFullName": "string (required)",
  "email": "string (required, valid email)",
  "password": "string (required, min 8 chars)",
  "phone": "string (required)",
  "businessType": "string (optional)",
  "website": "string (optional)",
  "country": "string (optional)"
}
```

**Expected Response 201:**
```json
{
  "message": "string",
  "data": {
    "companyId": "number",
    "companyName": "string",
    "userId": "number",
    "userEmail": "string",
    "userRole": "owner",
    "token": "string",
    "trialStartDate": "string",
    "trialEndDate": "string",
    "trialDaysRemaining": "number",
    "licenseKey": "string",
    "licenseType": "trial",
    "companyStatus": "trial"
  }
}
```

**Frontend stores after success:**
- `localStorage.token` = `data.token`
- `localStorage.company_id` = `data.companyId`
- `localStorage.user_role` = `data.userRole`
- `localStorage.trial_days` = `data.trialDaysRemaining`

**Frontend Impact:**
- `data.token` is mandatory — used for all subsequent API calls
- `data.companyId` is mandatory — injected in every request as `?company_id=`
- `data.userRole` must be `"owner"` — determines dashboard access level
- `data.companyStatus` must be `"trial"` — shown in trial banner

---

## POST /auth/login
**Purpose:** Login with email/password.
**Auth:** None
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "email": "string (required)",
  "password": "string (required)"
}
```

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "userId": "number",
    "userEmail": "string",
    "companyId": "number",
    "companyName": "string",
    "companyStatus": "trial | active | expired | suspended",
    "userRole": "owner | admin | manager | staff",
    "token": "string",
    "licenseKey": "string",
    "licenseType": "trial | paid",
    "trialDaysRemaining": "number (optional)",
    "subscriptionEndDate": "string (optional)"
  }
}
```

**Frontend stores after success:**
- `localStorage.token` = `data.token`
- `localStorage.company_id` = `data.companyId`
- `localStorage.user_role` = `data.userRole`
- `localStorage.trial_days` = `data.trialDaysRemaining`
- Cookie `token` = `data.token` (set for middleware)

**Frontend Impact:**
- If `data.token` is missing → login fails silently
- If `data.companyId` is missing → all subsequent requests will lack `company_id`
- `data.companyStatus` = `"expired"` or `"suspended"` → frontend redirects to `/dashboard/billing/blocked-access`
- `data.userRole` controls which nav items are visible

---

## POST /auth/logout
**Purpose:** Logout current user.
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Expected Response 200:**
```json
{ "message": "string", "data": { "success": true } }
```

**Frontend behavior:** Clears localStorage and cookie regardless of response.

---

## GET /auth/me
**Purpose:** Fetch current user + company on app load / refresh.
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "user": {
      "id": "number",
      "companyId": "number",
      "email": "string",
      "fullName": "string",
      "role": "owner | admin | manager | staff",
      "status": "active | invited | inactive",
      "joinedDate": "string",
      "lastLogin": "string (optional)"
    },
    "company": {
      "id": "number",
      "name": "string",
      "status": "trial | active | expired | suspended",
      "trialStartDate": "string",
      "trialEndDate": "string",
      "trialDaysRemaining": "number",
      "licenseKey": "string",
      "licenseType": "trial | paid"
    }
  }
}
```

**Frontend uses:**
- `data.user.role` → nav menu visibility
- `data.company.status` → trial banner, access blocking
- `data.company.trialDaysRemaining` → trial countdown UI

---

## POST /auth/forgot-password
**Endpoint called:** `POST /auth/password/forgot`
**Auth:** None
**Request Body:** `{ "email": "string" }`
**Expected Response 200:** `{ "message": "string", "data": { "resetTokenSent": true, "expiresIn": number } }`

---

## POST /auth/reset-password
**Endpoint called:** `POST /auth/password/reset`
**Auth:** None
**Request Body:**
```json
{
  "token": "string (from URL query param)",
  "newPassword": "string (min 8)",
  "confirmPassword": "string (min 8, must match)"
}
```
**Expected Response 200:** `{ "message": "string", "data": { "success": true, "redirectUrl": "string" } }`

---

## Frontend Validation Rules
| Field | Rule |
|-------|------|
| email | required, valid email format |
| password | required, min 8 characters |
| confirmPassword | must equal newPassword |
| companyName | required |
| ownerFullName | required |
| phone | required (signup) |

## Frontend Impact Notes
- The response envelope must be `{ message, data }` — frontend destructures `response.data.data`
- `token` in response data is stored immediately and used for all protected calls
- `company_id` must be a numeric type (parseInt applied on read)
- A `401` on any protected call will clear auth state and redirect to login
