# Task Scheduling Module

**Frontend Route:** `/schedule`
**API Base Path:** `/schedule`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interface

```typescript
interface ScheduleTask {
  id: string;
  title: string;
  description: string;
  category: "Planting" | "Harvesting" | "Irrigation" | "Feeding" | "Maintenance" | "Veterinary" | "Other";
  priority: "Low" | "Medium" | "High" | "Urgent";
  status: "Pending" | "In Progress" | "Completed" | "Overdue";
  assignee: string;           // Worker name or ID
  dueDate: string;            // ISO date string — "2024-04-10"
  startDate: string;          // ISO date string — "2024-04-08"
}

interface ScheduleStats {
  pending: number;
  inProgress: number;
  completed: number;
  overdue: number;
}
```

---

## Overdue Logic

A task is considered **overdue** when:
- `status` is NOT `"Completed"`, AND
- `dueDate` is before today's date

The backend should either:
1. Automatically update `status` to `"Overdue"` for qualifying tasks via a scheduled job (cron), or
2. Compute overdue status dynamically in query responses without changing the stored value

The frontend currently displays the `status` field as returned by the API. If a task has `status: "Pending"` but its `dueDate` is in the past, the frontend will not automatically show it as overdue unless the backend returns `"Overdue"`.

**Recommendation:** Use option 1 (background cron job) for consistency. Run hourly or daily.

---

## Endpoints

---

### GET `/schedule`

List all tasks with optional filters and pagination.

**Auth Required:** Yes

**Query Parameters:**

| Param      | Type    | Required | Description                                                       |
|------------|---------|----------|-------------------------------------------------------------------|
| `farm_id`  | string  | Yes      | Active farm UUID                                                  |
| `search`   | string  | No       | Partial match on `title`, `description`, or `assignee`           |
| `category` | string  | No       | Filter by: `Planting\|Harvesting\|Irrigation\|Feeding\|Maintenance\|Veterinary\|Other` |
| `status`   | string  | No       | Filter by: `Pending\|In Progress\|Completed\|Overdue`             |
| `priority` | string  | No       | Filter by: `Low\|Medium\|High\|Urgent`                            |
| `date`     | string  | No       | ISO date — filter tasks where `startDate <= date <= dueDate`      |
| `page`     | integer | No       | Page number, default `1`                                          |
| `limit`    | integer | No       | Page size, default `20`                                           |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 38,
    "page": 1,
    "limit": 20,
    "data": [
      {
        "id": "task-uuid-001",
        "title": "Apply irrigation to North Field",
        "description": "Run drip irrigation system for 4 hours",
        "category": "Irrigation",
        "priority": "High",
        "status": "Pending",
        "assignee": "Ahmed Karim",
        "dueDate": "2024-04-12",
        "startDate": "2024-04-10"
      }
    ]
  }
}
```

---

### POST `/schedule`

Create a new scheduled task.

**Auth Required:** Yes

**Request Body:**

```json
{
  "title": "Apply irrigation to North Field",
  "description": "Run drip irrigation system for 4 hours",
  "category": "Irrigation",
  "priority": "High",
  "status": "Pending",
  "assignee": "Ahmed Karim",
  "dueDate": "2024-04-12",
  "startDate": "2024-04-10",
  "farm_id": "farm-uuid-here"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Task created successfully",
  "data": { /* full ScheduleTask object with id */ }
}
```

**Validation Rules:**
- `title`: required, 1–200 characters
- `description`: optional, max 1000 characters
- `category`: required, must be one of the enum values
- `priority`: required, must be one of: `Low | Medium | High | Urgent`
- `status`: required, must be one of: `Pending | In Progress | Completed | Overdue`
- `assignee`: optional string, max 100 characters
- `dueDate`: required, valid ISO date
- `startDate`: optional, valid ISO date, must be <= `dueDate` if provided
- `farm_id`: required

---

### GET `/schedule/stats`

Returns task counts grouped by status.

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
    "pending": 12,
    "inProgress": 5,
    "completed": 18,
    "overdue": 3
  }
}
```

**Note:** Keys use camelCase: `inProgress` (not `in_progress` or `In Progress`). The frontend reads these exact keys for the stat cards.

**Important — Routing Note:** Register `/schedule/stats` **before** `/schedule/:id`.

---

### GET `/schedule/dates`

Returns an array of date strings on which tasks exist. Used by the calendar view to highlight days that have scheduled tasks.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                       |
|-----------|--------|----------|---------------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                                  |
| `month`   | string | No       | ISO month string: `"2024-04"` — filter to a month |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "dates": [
      "2024-04-08",
      "2024-04-10",
      "2024-04-12",
      "2024-04-15"
    ]
  }
}
```

**Logic:** Return all unique dates where a task exists (any task where the date falls between `startDate` and `dueDate`, or where `dueDate` equals the date). Dates should be ISO date strings (`YYYY-MM-DD`).

**Frontend Impact:** The calendar component calls this endpoint on month change to determine which day cells to mark with a dot/indicator. If this endpoint is slow or unavailable, the calendar will render without task indicators.

**Important — Routing Note:** Register `/schedule/dates` **before** `/schedule/:id`.

---

### GET `/schedule/:id`

Fetch a single task by ID.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "task-uuid-001",
    "title": "Apply irrigation to North Field",
    "description": "Run drip irrigation system for 4 hours",
    "category": "Irrigation",
    "priority": "High",
    "status": "Pending",
    "assignee": "Ahmed Karim",
    "dueDate": "2024-04-12",
    "startDate": "2024-04-10"
  }
}
```

---

### PUT `/schedule/:id`

Full update of a task record.

**Auth Required:** Yes

**Request Body:** Same shape as `POST /schedule` (all fields, without `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Task updated successfully",
  "data": { /* updated ScheduleTask object */ }
}
```

---

### PATCH `/schedule/:id/status`

Update only the status field of a task.

**Auth Required:** Yes

**Request Body:**

```json
{
  "status": "Completed"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Task status updated",
  "data": {
    "id": "task-uuid-001",
    "status": "Completed"
  }
}
```

**Validation Rules:**
- `status`: required, must be one of: `Pending | In Progress | Completed | Overdue`

---

### DELETE `/schedule/:id`

Delete a scheduled task.

**Auth Required:** Yes

**Success Response — `204 No Content`**

---

## Notes

- The `status` enum values use Title Case with spaces: `"In Progress"`, not `"in_progress"` or `"InProgress"`. The frontend uses these exact strings for display and comparison.
- The `priority` enum values also use Title Case: `"Low"`, `"Medium"`, `"High"`, `"Urgent"`.
- `assignee` is a free-form string in the current model. Future versions may link to the Workers module via a foreign key.
- The `date` filter on `GET /schedule` is used by the calendar view — when a user clicks a day, the frontend sends `?date=2024-04-10` to show tasks scheduled on or around that date.
- Consider adding a `completedAt` timestamp field for tracking when tasks were marked complete, useful for worker productivity reports.
- The `/schedule/dates` endpoint is a key enabler for calendar UX. It should be fast (ideally a simple DISTINCT query). Avoid making it do full pagination.
