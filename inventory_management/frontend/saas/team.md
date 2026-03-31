# Frontend Module: SaaS Team Management

## Pages
- `/dashboard/settings/team` — List team members, invite new users, manage roles
- `/dashboard/settings/team/:userId` — View individual user details

## API Service
`lib/saasCompanyApi.ts` (team methods are part of `saasCompanyApi`)

---

## GET /company/team/users
**Purpose:** Fetch all users belonging to the company, including invited (pending) users
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "users": [
      {
        "id": 1,
        "companyId": 1,
        "email": "user@example.com",
        "fullName": "string",
        "role": "admin | manager | staff",
        "roleId": 1,
        "staffRole": {
          "id": 1,
          "companyId": 1,
          "name": "string",
          "createdAt": "2024-01-01T00:00:00Z",
          "updatedAt": "2024-01-01T00:00:00Z"
        },
        "status": "active | invited | inactive",
        "joinedDate": "2024-01-01T00:00:00Z",
        "lastLogin": "2024-01-01T00:00:00Z",
        "avatar": "string"
      }
    ],
    "totalUsers": 3,
    "maxUsers": 10,
    "canAddMore": true
  }
}
```

**Frontend Impact:**
- Team list page — user table, showing status badges (active/invited/inactive)
- `canAddMore` gates the "Invite User" button
- `totalUsers` / `maxUsers` displayed as usage counter

---

## GET /company/team/users/:userId
**Purpose:** Fetch a single team member's details
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `userId` — numeric user ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "companyId": 1,
    "email": "user@example.com",
    "fullName": "string",
    "role": "admin | manager | staff",
    "roleId": 1,
    "staffRole": { "...StaffRole object or null..." },
    "status": "active | invited | inactive",
    "joinedDate": "2024-01-01T00:00:00Z",
    "lastLogin": "2024-01-01T00:00:00Z",
    "avatar": "string"
  }
}
```

**Frontend Impact:**
- User detail page / modal population

---

## POST /company/team/users/invite
**Purpose:** Send an invitation email and create a pending user record
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "email": "newuser@example.com",
  "fullName": "string",
  "roleId": 1
}
```

**Note:** All three fields are required. `roleId` must be a valid staff role ID for this company.

**Expected Response 201:**
```json
{
  "message": "string",
  "data": {
    "userId": 5,
    "email": "newuser@example.com",
    "status": "invited",
    "invitationToken": "string",
    "expiresAt": "2024-01-08T00:00:00Z",
    "invitedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Team list refreshed after successful invite
- Success toast shown with email address
- If `canAddMore` is false before calling, the button should be disabled (client-side guard)

---

## PATCH /company/team/users/:userId/role
**Purpose:** Change an existing team member's role
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `userId` — numeric user ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "role": "admin | manager | staff"
}
```

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "userId": 5,
    "role": "manager",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Role badge updated in the team list row without full reload
- Cannot demote your own account from admin (should be enforced by backend; frontend may add a guard)

---

## DELETE /company/team/users/:userId
**Purpose:** Remove a user from the company (revoke access)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `userId` — numeric user ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "userId": 5,
    "success": true,
    "removedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- User row removed from team list
- `totalUsers` counter decremented
- If removed user is currently logged in, their next request will return 401

---

## POST /company/team/users/resend-invitation/:userId
**Purpose:** Resend the invitation email to a pending user
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `userId` — numeric user ID (must have status `invited`)

**Query Params:**
- `company_id` (auto-injected)

**Request Body:** None

**Expected Response 200:**
```json
{
  "message": "Invitation resent successfully"
}
```

**Frontend Impact:**
- "Resend" button available only for users with `status: "invited"`
- Success toast confirms email resent

---

## POST /company/team/users/accept-invitation
**Purpose:** Accept a team invitation using the token from the invitation email
**Auth:** No auth required (called before user has a token)
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected — may not be available pre-login; check backend behavior)

**Request Body:**
```json
{
  "token": "invitation-token-string"
}
```

**Note:** This endpoint is called on the invitation acceptance page, typically at `/auth/accept-invitation?token=...`. The token is extracted from the URL query parameter and sent in the request body.

**Expected Response 200:**
```json
{
  "message": "string"
}
```

**Frontend Impact:**
- On success, redirects user to onboarding or login flow
- On token expiry/invalid, shows error page prompting to request a new invitation
