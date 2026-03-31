# Farm Workers / Staff Module

**Frontend Route:** `/workers`
**API Base Path:** `/workers`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interface

```typescript
interface Worker {
  id: string;
  name: string;
  role: "Farm Manager" | "Field Worker" | "Equipment Operator" | "Veterinarian" | "Irrigation Specialist" | "Harvester";
  phone: string;
  email: string;
  status: "active" | "on-leave" | "inactive";
  hireDate: string;           // ISO date string — "2022-01-15"
  dailyWage: number;          // Numeric, in local currency
  assignedArea: string;       // Free-form area/zone name, e.g. "North Field"
}

interface WorkerStats {
  total: number;
  active: number;
  onLeave: number;
  inactive: number;
}
```

---

## Endpoints

---

### GET `/workers`

List all workers for the active farm with optional filters and pagination.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type    | Required | Description                                                                 |
|-----------|---------|----------|-----------------------------------------------------------------------------|
| `farm_id` | string  | Yes      | Active farm UUID                                                            |
| `search`  | string  | No       | Partial match on `name`, `email`, or `phone`                                |
| `role`    | string  | No       | Filter by role: `Farm Manager\|Field Worker\|Equipment Operator\|Veterinarian\|Irrigation Specialist\|Harvester` |
| `status`  | string  | No       | Filter by status: `active\|on-leave\|inactive`                              |
| `page`    | integer | No       | Page number, default `1`                                                    |
| `limit`   | integer | No       | Page size, default `20`                                                     |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 24,
    "page": 1,
    "limit": 20,
    "data": [
      {
        "id": "worker-uuid-001",
        "name": "Maria Santos",
        "role": "Field Worker",
        "phone": "+254712345678",
        "email": "maria.santos@farm.com",
        "status": "active",
        "hireDate": "2022-01-15",
        "dailyWage": 1200,
        "assignedArea": "North Field"
      }
    ]
  }
}
```

---

### POST `/workers`

Add a new worker record.

**Auth Required:** Yes

**Request Body:**

```json
{
  "name": "Maria Santos",
  "role": "Field Worker",
  "phone": "+254712345678",
  "email": "maria.santos@farm.com",
  "status": "active",
  "hireDate": "2022-01-15",
  "dailyWage": 1200,
  "assignedArea": "North Field",
  "farm_id": "farm-uuid-here"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Worker added successfully",
  "data": { /* full Worker object with id */ }
}
```

**Validation Rules:**
- `name`: required, 2–100 characters
- `role`: required, must be one of the enum values (case-sensitive, spaces included)
- `phone`: optional, valid phone format (digits, spaces, `+`, `-`, `()` allowed)
- `email`: optional, valid email format; if provided must be unique within the farm
- `status`: required, one of: `active | on-leave | inactive`
- `hireDate`: optional, valid ISO date, must not be in the future
- `dailyWage`: optional, number >= 0
- `assignedArea`: optional string, max 100 characters
- `farm_id`: required

---

### GET `/workers/stats`

Returns counts of workers grouped by status.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description      |
|-----------|--------|----------|------------------|
| `farm_id` | string | Yes      | Active farm UUID |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 24,
    "active": 18,
    "onLeave": 4,
    "inactive": 2
  }
}
```

**Note:** The `onLeave` key uses camelCase (not `on_leave` or `on-leave`). The frontend reads this exact key.

**Important — Routing Note:** Register `/workers/stats` **before** `/workers/:id`.

---

### GET `/workers/:id`

Fetch a single worker by ID.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "worker-uuid-001",
    "name": "Maria Santos",
    "role": "Field Worker",
    "phone": "+254712345678",
    "email": "maria.santos@farm.com",
    "status": "active",
    "hireDate": "2022-01-15",
    "dailyWage": 1200,
    "assignedArea": "North Field"
  }
}
```

**Error Response:**

| Code  | Scenario      |
|-------|---------------|
| `404` | Worker not found |

---

### PUT `/workers/:id`

Full update of a worker record.

**Auth Required:** Yes

**Request Body:** Same shape as `POST /workers` (all fields, without `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Worker updated successfully",
  "data": { /* updated Worker object */ }
}
```

---

### PATCH `/workers/:id/status`

Update only the employment status of a worker.

**Auth Required:** Yes

**Request Body:**

```json
{
  "status": "on-leave"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Worker status updated",
  "data": {
    "id": "worker-uuid-001",
    "status": "on-leave"
  }
}
```

**Validation Rules:**
- `status`: required, must be one of: `active | on-leave | inactive`

**Note:** The status value in the request body and in the database uses hyphen notation: `"on-leave"` (not `"onLeave"` or `"on_leave"`). Ensure the backend stores and returns it consistently as `"on-leave"`.

---

### DELETE `/workers/:id`

Remove a worker from the farm.

**Auth Required:** Yes

**Success Response — `204 No Content`**

**Important:** Before deleting, consider whether the worker is referenced by any tasks in the Schedule module (`assignee` field) or shed assignments. The current data model stores `assignee` as a free-form string, so cascading is not an issue. If this changes to a foreign key in the future, implement `ON DELETE SET NULL`.

---

## Notes

- The `role` enum values use Title Case with spaces (e.g. `"Farm Manager"`, `"Equipment Operator"`). These exact strings are used in the frontend's role filter dropdown and role badge display.
- `dailyWage` is a number (not a string). Return it as a JSON number. The frontend uses it for payroll calculations and chart data in the Finance module.
- `assignedArea` is a free-form string. It does not reference the Field Map module's field IDs in the current model. If a relational link is needed, add a `fieldId` foreign key column separately.
- `email` uniqueness is scoped to the farm, not globally. A person could work on multiple farms with the same email.
- Consider adding a `photoUrl` field in a future iteration for worker profile photos.
- Workers are referenced by name in the Livestock Sheds module's `assignedWorker` field. If worker names change, shed assignments may become stale. A future improvement would be to use worker IDs as foreign keys.
