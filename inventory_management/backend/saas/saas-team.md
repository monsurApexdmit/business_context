# Module: SaaS Team

## Business Purpose
Manages team member access within a company. Owners can invite users via email (creating an Invitation record), update team member roles, and remove members. New members accept invitations via a token and set their password. Team size is enforced against subscription plan limits.

---

## Database Tables

### Table: `saas_users`
| Column      | Type          | Notes                                                           |
|-------------|---------------|-----------------------------------------------------------------|
| id          | uint (PK, AI) |                                                                 |
| company_id  | uint (FK)     | FK → companies.id                                               |
| email       | varchar(255)  | unique per company: idx_company_email                           |
| full_name   | string        | json: fullName                                                  |
| password    | string        | bcrypt hashed; json:"-" (never returned)                        |
| role        | string        | 'owner','admin','manager','staff'; json: role                   |
| role_id     | *uint (FK)    | FK → staff_roles.id; nullable; json: roleId                     |
| status      | string        | 'active','invited','inactive'; default 'active'; json: status   |
| joined_date | timestamp     | json: joinedDate                                                |
| last_login  | *timestamp    | nullable; json: lastLogin                                       |
| avatar      | string        | json: avatar                                                    |
| created_at  | timestamp     |                                                                 |
| updated_at  | timestamp     |                                                                 |

### Table: `invitations`
| Column           | Type          | Notes                                                      |
|------------------|---------------|------------------------------------------------------------|
| id               | uint (PK, AI) |                                                            |
| company_id       | uint (FK)     | FK → companies.id                                          |
| email            | varchar(255)  | json: email                                                |
| full_name        | string        | json: fullName                                             |
| role_id          | *uint (FK)    | FK → staff_roles.id; nullable; json: roleId                |
| status           | string        | 'pending','accepted','expired'; json: status               |
| invitation_token | varchar(255)  | unique; json: invitationToken                              |
| expires_at       | timestamp     | 7 days from invite; json: expiresAt                        |
| accepted_at      | *timestamp    | nullable; json: acceptedAt                                 |
| invited_at       | timestamp     | json: invitedAt                                            |

---

## Relationships
- SaasUser → StaffRole (BelongsTo via role_id, optional)
- Invitation → StaffRole (BelongsTo via role_id, preloaded on AcceptInvitation)

---

## Business Logic
- **Team limit**: checked against subscription plan's `max_users` (default 5 if no subscription).
- **InviteUser**: verifies role exists for company, checks email not already in team, checks team limit; creates Invitation with 7-day expiry and random 32-byte hex token.
- **UpdateUserRole**: cannot change own role; cannot change owner role; updates `role` field on SaasUser.
- **RemoveUser**: cannot remove self; cannot remove owner; soft-deletes SaasUser.
- **ResendInvitation**: generates new token, extends expiry by 7 days, updates invited_at.
- **AcceptInvitation**: finds pending invitation by token that hasn't expired; creates SaasUser with role from StaffRole.name, marks invitation as accepted; returns JWT token.
- Accepted user's role string is set from `invitation.Role.Name` (the StaffRole name, not a fixed enum value).

---

## Endpoints

### GET /auth/team
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Team members retrieved",
  "data": {
    "users": [
      {
        "id": 1,
        "email": "john@acme.com",
        "fullName": "John Doe",
        "role": "owner",
        "roleId": null,
        "staffRole": null,
        "status": "active",
        "joinedDate": "2024-01-01T00:00:00Z",
        "lastLogin": "2024-01-10T08:00:00Z",
        "avatar": ""
      }
    ],
    "totalUsers": 3,
    "maxUsers": 5,
    "canAddMore": true
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid or missing company ID"}`
- `500` — `{"message": "Failed to fetch team members"}`

---

### POST /auth/team/invite
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "email": "jane@acme.com",
  "fullName": "Jane Smith",
  "roleId": 2
}
```

**Validation:**
- `email` — required (from DTO binding)
- `fullName` — required
- `roleId` — required; must exist in company

**Response 201:**
```json
{
  "message": "Invitation sent successfully",
  "data": {
    "userId": 5,
    "email": "jane@acme.com",
    "status": "pending",
    "roleId": 2,
    "invitationToken": "abc123...",
    "expiresAt": "2024-01-08T00:00:00Z",
    "invitedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid or missing company ID"}` / `{"message": "Invalid request", "error": "..."}`
- `403` — `{"message": "Team member limit reached"}`
- `404` — `{"message": "Role not found for this company"}`
- `409` — `{"message": "User with this email already exists in the company"}`
- `500` — `{"message": "Failed to generate invitation token"}` / `{"message": "Failed to create invitation"}`

---

### PUT /auth/team/:userId/role
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:** `userId` (uint)

**Request Body:**
```json
{
  "role": "admin"
}
```

**Response 200:**
```json
{
  "message": "User role updated successfully",
  "data": {
    "userId": 3,
    "role": "admin",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid user ID"}` / `{"message": "Invalid or missing company ID"}` / `{"message": "Invalid current user"}` / `{"message": "Invalid request"}`
- `403` — `{"message": "Cannot change your own role"}` / `{"message": "Cannot change owner role"}`
- `404` — `{"message": "User not found"}`
- `500` — `{"message": "Failed to update user role"}`

---

### DELETE /auth/team/:userId
**Auth:** Bearer JWT required

**Path Params:** `userId` (uint)

**Response 200:**
```json
{
  "message": "User removed successfully",
  "data": {
    "userId": 3,
    "success": true,
    "removedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid user ID"}` / `{"message": "Invalid or missing company ID"}` / `{"message": "Invalid current user"}`
- `403` — `{"message": "Cannot remove yourself"}` / `{"message": "Cannot remove owner"}`
- `404` — `{"message": "User not found"}`
- `500` — `{"message": "Failed to remove user"}`

---

### POST /auth/team/:userId/resend-invitation
**Auth:** Bearer JWT required

**Path Params:** `userId` — the invitation ID

**Response 200:**
```json
{
  "message": "Invitation resent successfully",
  "data": null
}
```

**Error Responses:**
- `400` — `{"message": "Invalid user ID"}` / `{"message": "Invalid or missing company ID"}`
- `404` — `{"message": "Invitation not found"}`
- `500` — `{"message": "Failed to generate new token"}` / `{"message": "Failed to resend invitation"}`

---

### POST /auth/accept-invitation
**Auth:** None
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "token": "abc123...",
  "password": "newpassword123",
  "companyId": 1
}
```

**Validation:**
- `token` — required
- `password` — required; min 8 chars
- `companyId` — optional; if provided, must match invitation's company_id

**Response 201:**
```json
{
  "message": "Invitation accepted successfully",
  "data": {
    "userId": 5,
    "token": "<jwt>",
    "email": "jane@acme.com",
    "fullName": "Jane Smith",
    "role": "Manager",
    "roleId": 2,
    "companyId": 1,
    "joinedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid request", "error": "..."}` / `{"message": "Invalid or expired invitation"}` / `{"message": "Invalid role in invitation"}`
- `403` — `{"message": "Company ID does not match invitation"}`
- `500` — `{"message": "Failed to process password"}` / `{"message": "Failed to create user"}`

---

## Dependencies
- **staff-roles** — invitations and users reference staff_roles via role_id
- **saas-billing** — subscription plan provides max_users limit
