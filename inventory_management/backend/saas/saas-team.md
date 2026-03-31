# Module Overview

**Module Name:** SaaS Team
**Scope:** Backend — SaaS Platform
**Purpose:** Manages team member access within a company. Owners can invite users via email, update member roles, and remove members. Invited users accept their invitation via a token-based flow and set their own password. Team size is enforced against the subscription plan's user limit.

---

# Business Context

In a multi-tenant SaaS setup, each company has a team of users who collaborate within the platform. This module handles the full lifecycle of team membership: invitation, acceptance, role management, and removal. Invitations are time-limited (7-day expiry) and use a secure random token. The team size is gated against the subscription plan's `max_users` limit, defaulting to 5 if no subscription exists. Role assignments are driven by the `staff_roles` table (not a fixed enum), giving companies flexibility in defining roles.

---

# Functional Scope

## Included

- Listing all team members for the authenticated company, including capacity metadata
- Inviting new users via email by creating an `invitations` record with a 7-day expiry
- Resending an existing invitation with a refreshed token and extended expiry
- Accepting an invitation via token to create a `saas_users` record and receive a JWT
- Updating a team member's role (with ownership protection rules)
- Removing a team member via soft-delete (with ownership protection rules)
- Enforcing team size limits from the subscription plan

## Excluded

- Authentication / login for existing team members (handled by the auth module)
- Role definition and management (handled by the staff-roles module)
- Subscription plan management (handled by the saas-billing module)
- Email delivery for invitations (the token is returned in the API response; delivery is presumably handled by an external email service)

---

# Architecture Notes

- All endpoints (except `POST /auth/accept-invitation`) are scoped to `company_id` from the JWT context.
- `saas_users` uses GORM soft deletes for member removal.
- Invitation tokens are generated as random 32-byte hex strings.
- The accepted user's `role` string is set from the `StaffRole.Name` value on the related `staff_roles` record — it is not a fixed enum.
- The path parameter on `POST /auth/team/:userId/resend-invitation` is named `userId` but actually refers to an **invitation ID** — this is a known naming inconsistency.
- Team limit defaults to 5 if no subscription record exists for the company.
- Email uniqueness for `saas_users` is enforced per company via `idx_company_email`.

---

# Data Model

**Table: `saas_users`**

| Column        | Type             | Constraints                                   | Notes                                          |
|---------------|------------------|-----------------------------------------------|------------------------------------------------|
| `id`          | uint             | PK, Auto Increment                            |                                                |
| `company_id`  | uint             | FK → `companies.id`                           |                                                |
| `email`       | varchar(255)     | Unique per company (`idx_company_email`)       |                                                |
| `full_name`   | string           |                                               | JSON key: `fullName`                           |
| `password`    | string           | bcrypt hashed                                 | JSON key: `"-"` (never returned in responses)  |
| `role`        | string           |                                               | JSON key: `role`; e.g. `owner`, `admin`, `manager`, `staff` |
| `role_id`     | *uint (nullable) | FK → `staff_roles.id`                         | JSON key: `roleId`                             |
| `status`      | string           | Default `'active'`                            | JSON key: `status`; values: `active`, `invited`, `inactive` |
| `joined_date` | timestamp        |                                               | JSON key: `joinedDate`                         |
| `last_login`  | *timestamp       | Nullable                                      | JSON key: `lastLogin`                          |
| `avatar`      | string           |                                               | JSON key: `avatar`                             |
| `uploaded_by` | *uint (nullable) | FK → `saas_users.id`                          | JSON key: `uploadedBy`; who uploaded the avatar |
| `created_at`  | timestamp        | Auto-managed                                  |                                                |
| `updated_at`  | timestamp        | Auto-managed                                  |                                                |
| `deleted_at`  | timestamp        | Soft delete                                   |                                                |

---

**Table: `invitations`**

| Column              | Type             | Constraints         | Notes                                          |
|---------------------|------------------|---------------------|------------------------------------------------|
| `id`                | uint             | PK, Auto Increment  |                                                |
| `company_id`        | uint             | FK → `companies.id` |                                                |
| `email`             | varchar(255)     |                     | JSON key: `email`                              |
| `full_name`         | string           |                     | JSON key: `fullName`                           |
| `role_id`           | *uint (nullable) | FK → `staff_roles.id` | JSON key: `roleId`                           |
| `status`            | string           |                     | JSON key: `status`; values: `pending`, `accepted`, `expired` |
| `invitation_token`  | varchar(255)     | Unique              | JSON key: `invitationToken`                    |
| `expires_at`        | timestamp        |                     | JSON key: `expiresAt`; set to 7 days from invite |
| `accepted_at`       | *timestamp       | Nullable            | JSON key: `acceptedAt`                         |
| `invited_at`        | timestamp        |                     | JSON key: `invitedAt`                          |

**Relationships:**

| Relationship                     | Type      | Key       | Notes                                         |
|----------------------------------|-----------|-----------|-----------------------------------------------|
| SaasUser → StaffRole             | BelongsTo | `role_id` | Optional                                      |
| Invitation → StaffRole           | BelongsTo | `role_id` | Preloaded during `AcceptInvitation`            |

---

# API Endpoints

| Method | URL                                     | Auth       | Purpose                                            |
|--------|-----------------------------------------|------------|----------------------------------------------------|
| GET    | `/auth/team`                            | Bearer JWT | List all team members with capacity metadata       |
| POST   | `/auth/team/invite`                     | Bearer JWT | Invite a new user by email                         |
| PUT    | `/auth/team/:userId/role`               | Bearer JWT | Update a team member's role                        |
| DELETE | `/auth/team/:userId`                    | Bearer JWT | Remove a team member (soft delete)                 |
| POST   | `/auth/team/:userId/resend-invitation`  | Bearer JWT | Resend an invitation with a new token and expiry   |
| POST   | `/auth/accept-invitation`               | None       | Accept an invitation and create a user account     |

---

# Request Payloads

**POST `/auth/team/invite` — Request Body**

```json
{
  "email":    "string (required)",
  "fullName": "string (required)",
  "roleId":   "uint (required)"
}
```

---

**PUT `/auth/team/:userId/role` — Path Param + Request Body**

- Path: `userId` — uint, the ID of the target `saas_users` record

```json
{
  "role": "string (required)"
}
```

---

**DELETE `/auth/team/:userId` — Path Param**

- Path: `userId` — uint, the ID of the `saas_users` record to remove
- No request body.

---

**POST `/auth/team/:userId/resend-invitation` — Path Param**

- Path: `userId` — uint. **Note:** despite the name, this is the **invitation ID**, not the user ID.
- No request body.

---

**POST `/auth/accept-invitation` — Request Body**

```json
{
  "token":     "string (required)",
  "password":  "string (required, min 8 characters)",
  "companyId": "uint (optional)"
}
```

---

# Response Contracts

**GET `/auth/team`** — 200 OK

```json
{
  "message": "Team members retrieved",
  "data": {
    "users": [
      {
        "id":         1,
        "email":      "john@acme.com",
        "fullName":   "John Doe",
        "role":       "owner",
        "roleId":     null,
        "staffRole":  null,
        "status":     "active",
        "joinedDate": "2024-01-01T00:00:00Z",
        "lastLogin":  "2024-01-10T08:00:00Z",
        "avatar":     ""
      }
    ],
    "totalUsers": 3,
    "maxUsers":   5,
    "canAddMore": true
  }
}
```

---

**POST `/auth/team/invite`** — 201 Created

```json
{
  "message": "Invitation sent successfully",
  "data": {
    "userId":           5,
    "email":            "jane@acme.com",
    "status":           "pending",
    "roleId":           2,
    "invitationToken":  "abc123...",
    "expiresAt":        "2024-01-08T00:00:00Z",
    "invitedAt":        "2024-01-01T00:00:00Z"
  }
}
```

---

**PUT `/auth/team/:userId/role`** — 200 OK

```json
{
  "message": "User role updated successfully",
  "data": {
    "userId":    3,
    "role":      "admin",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

---

**DELETE `/auth/team/:userId`** — 200 OK

```json
{
  "message": "User removed successfully",
  "data": {
    "userId":    3,
    "success":   true,
    "removedAt": "2024-01-01T00:00:00Z"
  }
}
```

---

**POST `/auth/team/:userId/resend-invitation`** — 200 OK

```json
{
  "message": "Invitation resent successfully",
  "data": null
}
```

---

**POST `/auth/accept-invitation`** — 201 Created

```json
{
  "message": "Invitation accepted successfully",
  "data": {
    "userId":    5,
    "token":     "<jwt>",
    "email":     "jane@acme.com",
    "fullName":  "Jane Smith",
    "role":      "Manager",
    "roleId":    2,
    "companyId": 1,
    "joinedAt":  "2024-01-01T00:00:00Z"
  }
}
```

---

# Validation Rules

| Field / Context           | Rule                                                                                  |
|---------------------------|---------------------------------------------------------------------------------------|
| `email` (invite)          | Required; must not already exist in `saas_users` for this company                    |
| `fullName` (invite)       | Required                                                                              |
| `roleId` (invite)         | Required; must reference an existing `staff_roles` record for the company             |
| `userId` (path param)     | Must be a valid uint; returns 400 if unparseable                                     |
| `role` (update role)      | Required in request body                                                              |
| `token` (accept)          | Required; invitation must be in `pending` status and not expired                     |
| `password` (accept)       | Required; minimum 8 characters                                                        |
| `companyId` (accept)      | Optional; if provided, must match the `company_id` on the invitation record           |
| Team limit                | Active team member count must be below `max_users` from subscription plan (default 5)|

---

# Business Logic Details

### InviteUser (POST `/auth/team/invite`)

1. Validate `company_id` from JWT.
2. Look up the `staff_roles` record by `roleId` scoped to the company — return 404 if not found.
3. Check that the email does not already exist in `saas_users` for this company — return 409 if duplicate.
4. Count active team members; compare against subscription plan's `max_users` — return 403 if limit reached.
5. Generate a random 32-byte hex token.
6. Create an `invitations` record with `status = 'pending'`, `expires_at = now + 7 days`.

### AcceptInvitation (POST `/auth/accept-invitation`)

1. Find the `invitations` record by `token` where `status = 'pending'` and `expires_at > now`.
2. If `companyId` is provided in the request, verify it matches `invitation.company_id` — return 403 if mismatch.
3. Preload `StaffRole` on the invitation; derive the user's `role` string from `invitation.Role.Name`.
4. Hash the provided password with bcrypt.
5. Create a `saas_users` record with the invitation data and the derived role.
6. Mark the invitation as `accepted` and set `accepted_at`.
7. Return a JWT token for the new user.

### UpdateUserRole (PUT `/auth/team/:userId/role`)

- A user cannot change their own role (returns 403).
- A user with `role = 'owner'` cannot have their role changed (returns 403).
- Applies the new role string directly to the `saas_users` record.

### RemoveUser (DELETE `/auth/team/:userId`)

- A user cannot remove themselves (returns 403).
- A user with `role = 'owner'` cannot be removed (returns 403).
- Soft-deletes the `saas_users` record.

### ResendInvitation (POST `/auth/team/:userId/resend-invitation`)

- Looks up the `invitations` record by the path parameter (invitation ID, not user ID).
- Generates a new 32-byte hex token.
- Extends `expires_at` by 7 days from now.
- Updates `invited_at` to the current timestamp.

---

# Dependencies

| Module / Table  | Relationship                                                                      |
|-----------------|-----------------------------------------------------------------------------------|
| `staff-roles`   | `invitations.role_id` and `saas_users.role_id` reference `staff_roles.id`        |
| `saas-billing`  | Subscription plan provides `max_users` limit enforced during invite               |
| `companies`     | `company_id` FK; all team data is scoped to a company                             |

---

# Security / Permissions

- All endpoints except `POST /auth/accept-invitation` require a valid Bearer JWT token.
- `company_id` is always resolved from JWT context — users cannot access or modify team data from other companies.
- Passwords are stored as bcrypt hashes and are never returned in any API response (`json:"-"`).
- Owner protection: the owner role cannot be changed or removed by any team action in this module.
- Self-protection: a user cannot remove themselves or change their own role.
- Invitation tokens are 32-byte random hex strings (256 bits of entropy) — they are single-use and expire after 7 days.

---

# Error Handling

| HTTP Status | Endpoint                              | Trigger                                                                     |
|-------------|---------------------------------------|-----------------------------------------------------------------------------|
| 400         | All authenticated endpoints           | Invalid or missing `company_id` in JWT; invalid `userId` path param         |
| 400         | POST `/auth/team/invite`              | Invalid request body                                                        |
| 400         | PUT `/auth/team/:userId/role`         | Invalid request body; invalid current user context                          |
| 400         | POST `/auth/accept-invitation`        | Invalid request body; expired or already-accepted token; invalid role in invitation |
| 403         | POST `/auth/team/invite`              | Team member limit reached                                                   |
| 403         | PUT `/auth/team/:userId/role`         | Cannot change own role; cannot change owner's role                          |
| 403         | DELETE `/auth/team/:userId`           | Cannot remove self; cannot remove owner                                     |
| 403         | POST `/auth/accept-invitation`        | Provided `companyId` does not match invitation's `company_id`               |
| 404         | POST `/auth/team/invite`              | Role not found for this company                                             |
| 404         | PUT `/auth/team/:userId/role`         | User not found                                                              |
| 404         | DELETE `/auth/team/:userId`           | User not found                                                              |
| 404         | POST `/auth/team/:userId/resend-invitation` | Invitation not found                                                  |
| 409         | POST `/auth/team/invite`              | Email already exists in the company's team                                  |
| 500         | All                                   | Internal server errors (DB failure, token generation failure, password hashing failure) |

**Error response shape:**

```json
{ "message": "Descriptive error message" }
```

With optional `error` field for detail:

```json
{ "message": "Invalid request", "error": "..." }
```

---

# Testing Notes

- Verify invite flow end-to-end: invite → token returned → accept with token → `saas_users` record created → JWT returned.
- Verify that accepting an expired invitation returns 400 (not 404).
- Verify that accepting an already-accepted invitation returns 400.
- Verify team limit enforcement: inviting when at `max_users` returns 403.
- Verify the default limit of 5 when no subscription exists.
- Test role-change protection: cannot change own role; cannot change owner's role.
- Test remove protection: cannot remove self; cannot remove owner.
- Verify resend generates a new token and extends `expires_at` by 7 days from the call time.
- Test `companyId` mismatch on accept-invitation returns 403.
- Verify that the `userId` path param on resend-invitation is treated as an invitation ID, not a user ID.
- Verify `password` is never returned in any response body.
- Test email uniqueness constraint: inviting an email that already exists in `saas_users` returns 409.

---

# Open Questions / Missing Details

- **Path parameter naming:** `POST /auth/team/:userId/resend-invitation` uses `:userId` but the value is an invitation ID. This inconsistency should be resolved in a future API revision.
- **Email delivery:** The invitation token is returned in the API response body. There is no documentation on how the invitation email is sent to the invitee (email service, template, etc.).
- **`status = 'invited'` in saas_users:** It is not documented whether a `saas_users` record with `status = 'invited'` is created at invite time, or whether the `invitations` table is the only record until acceptance.
- **Role string vs. role_id:** The `saas_users` table has both a `role` (string) and a `role_id` (FK). After acceptance, `role` is set from `StaffRole.Name`. It is unclear which field is authoritative for permission checks.
- **Expiry handling:** There is no documented background job or endpoint to mark `invitations` records as `expired` when `expires_at` passes.
- **Owner invitation/creation:** It is not documented how the initial owner (`role = 'owner'`) record in `saas_users` is created (presumably during company registration).
- **Soft-delete field:** The soft-delete column on `saas_users` is not explicitly listed in the schema. Its presence is implied by the remove logic.

---

# Change Log / Notes

| Date       | Note                                                                   |
|------------|------------------------------------------------------------------------|
| 2026-03-31 | Initial standardized documentation pass from source notes              |
