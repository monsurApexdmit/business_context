# Notifications Module

**Frontend Route:** `/notifications`
**API Base Path:** `/notifications`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interface

```typescript
interface Notification {
  id: string;
  title: string;
  message: string;
  type: "weather" | "inventory" | "task" | "system";
  severity: "info" | "warning" | "critical";
  createdAt: string;          // ISO datetime string — "2024-03-15T09:30:00Z"
  read: boolean;
}
```

---

## Notification Sources

Notifications are created server-side (not by the frontend). The backend should auto-generate notifications based on:

| Type        | Trigger Condition                                                     |
|-------------|-----------------------------------------------------------------------|
| `inventory` | Inventory item quantity drops to or below `minStock`                  |
| `inventory` | Inventory item quantity reaches `0` (out-of-stock)                    |
| `task`      | A scheduled task's `dueDate` has passed and status is not `Completed` |
| `task`      | A task with `priority: "Urgent"` is due within 24 hours               |
| `weather`   | External weather service integration (future scope)                   |
| `system`    | System maintenance, account changes, etc.                             |

The frontend does not create notifications directly. Notifications are managed by backend automated logic.

---

## Endpoints

---

### GET `/notifications`

List all notifications for the authenticated user's active farm.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type    | Required | Description                                         |
|-----------|---------|----------|-----------------------------------------------------|
| `farm_id` | string  | Yes      | Active farm UUID                                    |
| `type`    | string  | No       | Filter by type: `weather\|inventory\|task\|system`  |
| `read`    | boolean | No       | Filter by read status: `true` or `false`            |
| `page`    | integer | No       | Page number, default `1`                            |
| `limit`   | integer | No       | Page size, default `30`                             |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 45,
    "page": 1,
    "limit": 30,
    "unreadCount": 8,
    "data": [
      {
        "id": "notif-uuid-001",
        "title": "Low Stock Alert",
        "message": "NPK Fertilizer is running low — only 12 bags remaining (minimum: 50)",
        "type": "inventory",
        "severity": "warning",
        "createdAt": "2024-03-15T09:30:00Z",
        "read": false
      },
      {
        "id": "notif-uuid-002",
        "title": "Task Overdue",
        "message": "Irrigation task for North Field was due on March 12 and is still pending",
        "type": "task",
        "severity": "critical",
        "createdAt": "2024-03-13T08:00:00Z",
        "read": false
      },
      {
        "id": "notif-uuid-003",
        "title": "System Update",
        "message": "The system was updated to version 2.1.0 on March 10",
        "type": "system",
        "severity": "info",
        "createdAt": "2024-03-10T06:00:00Z",
        "read": true
      }
    ]
  }
}
```

**Note:** The `unreadCount` field in the response envelope allows the frontend to update the notification badge count without needing a separate API call.

**Default sort order:** `createdAt DESC` (most recent notifications first).

---

### GET `/notifications/unread-count`

Returns only the count of unread notifications. Used for the notification bell badge in the navigation header.

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
    "count": 8
  }
}
```

**Frontend Impact:** This endpoint is polled on a regular interval (e.g. every 60 seconds) or on page focus to keep the notification badge up to date. It must be a lightweight, fast query. Avoid joining large tables.

**Important — Routing Note:** Register `/notifications/unread-count` and `/notifications/read-all` **before** `/notifications/:id` to prevent routing conflicts.

---

### PATCH `/notifications/:id/read`

Mark a single notification as read.

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description       |
|-------|--------|-------------------|
| `id`  | string | Notification UUID |

**Request Body:**

```json
{
  "read": true
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Notification marked as read",
  "data": {
    "id": "notif-uuid-001",
    "read": true
  }
}
```

**Note:** `read: false` in the request body would mark a notification as unread (re-open). The frontend currently only sends `{ read: true }`, but support for `false` is included for completeness.

**Validation Rules:**
- `read`: required, boolean

---

### PATCH `/notifications/read-all`

Mark all unread notifications for the farm as read.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description      |
|-----------|--------|----------|------------------|
| `farm_id` | string | Yes      | Active farm UUID |

**Request Body:** None required. The farm is scoped via `farm_id` query param and the authenticated user's token.

**Success Response — `200 OK`:**

```json
{
  "message": "All notifications marked as read",
  "data": {
    "updatedCount": 8
  }
}
```

`updatedCount` is the number of notifications that were updated from `read: false` to `read: true`.

**Important — Routing Note:** This path (`/notifications/read-all`) is a static path and must be registered before `/notifications/:id` in the router. Otherwise Express/other routers will try to find a notification with ID `"read-all"`.

---

### DELETE `/notifications/:id`

Dismiss and permanently delete a single notification.

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description       |
|-------|--------|-------------------|
| `id`  | string | Notification UUID |

**Success Response — `204 No Content`**

**Error Responses:**

| Code  | Scenario                         |
|-------|----------------------------------|
| `404` | Notification not found           |
| `403` | Notification belongs to a different farm |

---

## Notification Severity Styles

The frontend renders notifications with visual styling based on `severity`:

| Severity   | Display Style                   |
|------------|----------------------------------|
| `info`     | Blue icon, neutral background    |
| `warning`  | Yellow/orange icon, amber tint   |
| `critical` | Red icon, red tint, bold title   |

Do not rename or add severity values without updating the frontend's style mapping.

---

## Notes

- Notifications should be **scoped per farm**, not per user. All users with access to a farm see the same notifications.
- Consider adding a `createdBy` field to system/task notifications to indicate which user triggered the action (e.g. "John Doe updated task X").
- If real-time notifications are required in the future, implement Server-Sent Events (SSE) or WebSockets for a push-based model. For now, polling via `GET /notifications/unread-count` is the intended pattern.
- Old notifications should be periodically cleaned up. Consider a retention policy (e.g. auto-delete notifications older than 30 days).
- When `inventory` notifications are generated, include the item name, current quantity, and minimum threshold in the `message` so the user can act without navigating to the inventory page.
- The `read` field defaults to `false` when a notification is created.
