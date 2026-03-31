# API Architecture — Agri/Farm Management System

## Overview

This document defines the global API design conventions that ALL backend modules must follow. The frontend is a React (Vite + React Router) SPA. There is currently no backend; this document serves as the contract the backend must implement.

---

## Base URL

The API base URL is configured via an environment variable:

```env
VITE_API_BASE_URL=https://api.yourdomain.com/api/v1
```

All endpoint paths in this documentation are relative to this base URL.

---

## Authentication

All protected routes require a Bearer token in the `Authorization` header:

```http
Authorization: Bearer <jwt_token>
```

The frontend stores the JWT token in `localStorage` under the key `token` after a successful login.

**Token lifecycle:**
- On `401 Unauthorized` response from any endpoint, the frontend clears `localStorage.token`, `localStorage.farm_id`, and `localStorage.user_role`, then redirects the user to `/signin`.
- Token must be validated server-side on every protected request.

Public (unauthenticated) endpoints:
- `POST /auth/signin`
- `POST /auth/signup`
- `POST /auth/forgot-password`
- `POST /auth/reset-password`

---

## Multi-Tenancy

The system is multi-tenant. Each authenticated user belongs to one or more farms. The active farm is passed as a query parameter on all data requests:

```
?farm_id=<uuid>
```

The frontend reads `farm_id` from `localStorage.farm_id` and appends it to every request automatically.

**Backend requirement:** Validate that the authenticated user has access to the requested `farm_id`. Return `403 Forbidden` if not.

---

## Content-Type

All request and response bodies use JSON unless explicitly noted otherwise:

```http
Content-Type: application/json
```

The only exception is the profile photo upload endpoint, which uses `multipart/form-data`. This is noted explicitly in the Settings module documentation.

---

## Standard Response Envelope

All API responses (success and error) must be wrapped in the following envelope:

### Success Response

```json
{
  "message": "Operation completed successfully",
  "data": { }
}
```

- `message`: Human-readable status string.
- `data`: The actual payload — an object or array depending on the endpoint.

### Error Response

```json
{
  "message": "Validation failed",
  "errors": [
    { "field": "email", "message": "Email is required" }
  ]
}
```

- `message`: Top-level error description.
- `errors` (optional): Array of field-level validation errors. Used by the frontend to display inline form errors.

---

## Pagination

All list endpoints that return potentially large datasets support pagination via query parameters:

**Request:**
```
GET /crops?page=1&limit=20&farm_id=abc123
```

| Query Param | Type    | Default | Description                         |
|-------------|---------|---------|-------------------------------------|
| `page`      | integer | `1`     | Page number, 1-indexed              |
| `limit`     | integer | `20`    | Number of items per page (max: 100) |

**Response shape for paginated lists:**

```json
{
  "message": "Success",
  "data": {
    "total": 84,
    "page": 1,
    "limit": 20,
    "data": [ ]
  }
}
```

| Field   | Type    | Description                          |
|---------|---------|--------------------------------------|
| `total` | integer | Total number of records matching the filter |
| `page`  | integer | Current page                         |
| `limit` | integer | Page size                            |
| `data`  | array   | Array of records for the current page |

---

## HTTP Status Codes

| Code  | Meaning                                                      |
|-------|--------------------------------------------------------------|
| `200` | OK — successful GET, PUT, PATCH                              |
| `201` | Created — successful POST                                    |
| `204` | No Content — successful DELETE (no body)                     |
| `400` | Bad Request — validation error, malformed body               |
| `401` | Unauthorized — missing or expired token                      |
| `403` | Forbidden — authenticated but not allowed (wrong farm, role) |
| `404` | Not Found — resource does not exist                          |
| `409` | Conflict — duplicate entry (e.g. duplicate tagId)            |
| `422` | Unprocessable Entity — semantic validation error             |
| `500` | Internal Server Error                                        |

---

## Common Filter Query Parameters

Most list endpoints support a consistent set of filter/search parameters in addition to pagination:

| Param    | Type   | Description                                         |
|----------|--------|-----------------------------------------------------|
| `search` | string | Full-text or partial match on name/title/identifier |
| `page`   | int    | Pagination page number                              |
| `limit`  | int    | Pagination page size                                |
| `farm_id`| string | Required for tenant isolation                       |

Module-specific filters (e.g. `status`, `category`, `type`) are documented per module.

---

## Sorting (Recommended)

While not strictly required by the current frontend, the backend should support optional sorting:

```
?sort_by=createdAt&sort_order=desc
```

This will allow the frontend to be extended without breaking changes.

---

## Modules

The following modules are documented in the `modules/` directory:

| Module                | File                             | Base Path             |
|-----------------------|----------------------------------|-----------------------|
| Authentication        | `../auth.md`                     | `/auth`               |
| Crop Management       | `modules/crops.md`               | `/crops`              |
| Individual Livestock  | `modules/livestock.md`           | `/livestock`          |
| Livestock Sheds       | `modules/livestock-sheds.md`     | `/sheds`              |
| Livestock Inventory   | `modules/livestock-inventory.md` | `/livestock-inventory`|
| Farm Supplies         | `modules/inventory.md`           | `/inventory`          |
| Task Scheduling       | `modules/schedule.md`            | `/schedule`           |
| Farm Workers          | `modules/workers.md`             | `/workers`            |
| Field/Plot Map        | `modules/field-map.md`           | `/fields`             |
| Financial Overview    | `modules/finances.md`            | `/finances`           |
| Reports & Analytics   | `modules/reports.md`             | `/reports`            |
| Notifications         | `modules/notifications.md`       | `/notifications`      |
| Settings              | `modules/settings.md`            | `/settings`           |

---

## Frontend HTTP Client Pattern

The frontend uses a central Axios (or Fetch) instance configured with:

1. `baseURL` from `VITE_API_BASE_URL`
2. Request interceptor — injects `Authorization: Bearer <token>` on every request
3. Request interceptor — injects `?farm_id=<id>` as a default query param
4. Response interceptor — on `401`, clears localStorage and redirects to `/signin`

Any change to auth header format or token structure will break ALL protected endpoints simultaneously.

---

## Date Format

All date fields must be ISO 8601 strings:

```
"2024-03-15"           // date only
"2024-03-15T10:30:00Z" // datetime with UTC timezone
```

The frontend renders these using `new Date(isoString)` and formats them via `toLocaleDateString()`. Do not return timestamps as Unix epoch integers unless explicitly noted.

---

## ID Format

All resource IDs should be UUIDs (v4):

```
"id": "550e8400-e29b-41d4-a716-446655440000"
```

The frontend treats all IDs as opaque strings. Do not return integer IDs unless the entire system is redesigned.

---

## CORS

The backend must allow cross-origin requests from the frontend's origin:

```
Access-Control-Allow-Origin: <frontend_origin>
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
```

---

## Rate Limiting (Recommended)

For production, implement per-IP or per-user rate limiting with appropriate headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1711234567
```

Return `429 Too Many Requests` when exceeded.
