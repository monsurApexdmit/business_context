# Livestock Shed Management Module

**Frontend Route:** `/livestock-sheds`
**API Base Path:** `/sheds`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interfaces

```typescript
interface Shed {
  id: string;
  name: string;
  animalType: "cow" | "sheep" | "chicken";
  totalAnimals: number;
  capacity: number;
  status: "active" | "maintenance" | "inactive";
  feedSchedule: string;       // Free-form description, e.g. "Twice daily at 7AM and 5PM"
  healthStatus: string;       // Free-form health note, e.g. "All healthy, vaccination due"
  cleaningStatus: string;     // Free-form, e.g. "Cleaned yesterday"
  productionData: string;     // Free-form, e.g. "120L/day milk"
  assignedWorker: string;     // Worker name or ID reference
  notes: string;
  gridPosition: {
    row: number;              // 0-indexed grid row
    col: number;              // 0-indexed grid column
    rowSpan: number;          // Number of rows the shed occupies
    colSpan: number;          // Number of columns the shed occupies
  };
}

interface ShedDailyRecord {
  id: string;
  shedId: string;
  type: "feeding" | "vaccination" | "treatment" | "cleaning" | "mortality";
  date: string;               // ISO date string
  details: string;
  createdAt: string;          // ISO datetime string
}

interface ShedStats {
  totalSheds: number;
  totalAnimals: number;
  capacityPercent: number;    // e.g. 78 (for 78%)
  activeSheds: number;
  alerts: {
    fullSheds: number;        // Sheds at 100% capacity
    sickAnimals: number;      // Count of sheds with health alerts
    feedAlerts: number;       // Count of sheds with overdue feeding
    cleaningDue: number;      // Count of sheds with cleaning overdue
  };
}
```

---

## Grid Layout Notes

The shed management page renders sheds on a **CSS Grid**. Each shed occupies a rectangular area defined by `gridPosition`. The grid dimensions are determined by the frontend layout (typically 4–6 columns).

- `row` and `col` are 0-indexed.
- `rowSpan` and `colSpan` define how many grid cells the shed card spans.
- When creating or moving a shed, the frontend sends the new grid position to the backend.
- The backend must persist `gridPosition` as a nested object (or as four separate columns) and return it in the exact nested format shown above.

---

## Endpoints

---

### GET `/sheds`

List all sheds for the active farm.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description      |
|-----------|--------|----------|------------------|
| `farm_id` | string | Yes      | Active farm UUID |

**Note:** This endpoint does **not** use standard pagination — the frontend renders all sheds at once in a grid layout. All sheds for the farm are returned in a single response.

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    {
      "id": "shed-uuid-001",
      "name": "Cow Shed A",
      "animalType": "cow",
      "totalAnimals": 45,
      "capacity": 60,
      "status": "active",
      "feedSchedule": "Twice daily at 7AM and 5PM",
      "healthStatus": "All healthy",
      "cleaningStatus": "Cleaned yesterday",
      "productionData": "120L/day milk",
      "assignedWorker": "John Mwangi",
      "notes": "Ventilation improved last week",
      "gridPosition": {
        "row": 0,
        "col": 0,
        "rowSpan": 2,
        "colSpan": 2
      }
    }
  ]
}
```

**Frontend Impact:** The list is returned as a flat array (not paginated). The frontend maps over the array and places each shed in the grid using its `gridPosition`. If `gridPosition` is missing or null, the shed will not render correctly.

---

### POST `/sheds`

Create a new shed.

**Auth Required:** Yes

**Request Body:**

```json
{
  "name": "Cow Shed A",
  "animalType": "cow",
  "totalAnimals": 45,
  "capacity": 60,
  "status": "active",
  "feedSchedule": "Twice daily at 7AM and 5PM",
  "healthStatus": "All healthy",
  "cleaningStatus": "Cleaned yesterday",
  "productionData": "120L/day milk",
  "assignedWorker": "John Mwangi",
  "notes": "",
  "gridPosition": {
    "row": 0,
    "col": 0,
    "rowSpan": 2,
    "colSpan": 2
  },
  "farm_id": "farm-uuid-here"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Shed created successfully",
  "data": { /* full Shed object with id */ }
}
```

**Validation Rules:**
- `name`: required, 1–100 characters, unique within the farm
- `animalType`: required, one of: `cow | sheep | chicken`
- `totalAnimals`: required, integer >= 0
- `capacity`: required, integer > 0, must be >= `totalAnimals`
- `status`: required, one of: `active | maintenance | inactive`
- `feedSchedule`: optional string
- `healthStatus`: optional string
- `cleaningStatus`: optional string
- `productionData`: optional string
- `assignedWorker`: optional string
- `notes`: optional string, max 1000 characters
- `gridPosition.row`: required, integer >= 0
- `gridPosition.col`: required, integer >= 0
- `gridPosition.rowSpan`: required, integer >= 1
- `gridPosition.colSpan`: required, integer >= 1
- `farm_id`: required

---

### GET `/sheds/stats`

Returns aggregated statistics and alert counts for all sheds in the active farm.

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
    "totalSheds": 8,
    "totalAnimals": 320,
    "capacityPercent": 78,
    "activeSheds": 7,
    "alerts": {
      "fullSheds": 2,
      "sickAnimals": 1,
      "feedAlerts": 0,
      "cleaningDue": 3
    }
  }
}
```

**Alert Definitions (backend logic):**
- `fullSheds`: count of sheds where `totalAnimals >= capacity`
- `sickAnimals`: count of sheds whose `healthStatus` contains keywords indicating sickness (e.g. "sick", "treatment") — or alternatively, maintain a boolean flag
- `feedAlerts`: business logic TBD — suggest a `lastFeedTime` field on Shed to enable this
- `cleaningDue`: business logic TBD — suggest a `lastCleanedDate` field on Shed to enable this

**Important — Routing Note:** Register `/sheds/stats` **before** `/sheds/:id` to prevent routing conflict.

---

### GET `/sheds/:id`

Fetch a single shed with its daily records.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "shed-uuid-001",
    "name": "Cow Shed A",
    "animalType": "cow",
    "totalAnimals": 45,
    "capacity": 60,
    "status": "active",
    "feedSchedule": "Twice daily at 7AM and 5PM",
    "healthStatus": "All healthy",
    "cleaningStatus": "Cleaned yesterday",
    "productionData": "120L/day milk",
    "assignedWorker": "John Mwangi",
    "notes": "",
    "gridPosition": {
      "row": 0,
      "col": 0,
      "rowSpan": 2,
      "colSpan": 2
    },
    "dailyRecords": [
      {
        "id": "record-uuid-001",
        "shedId": "shed-uuid-001",
        "type": "feeding",
        "date": "2024-03-15",
        "details": "Morning feeding completed — 200kg hay distributed",
        "createdAt": "2024-03-15T07:30:00Z"
      }
    ]
  }
}
```

---

### PUT `/sheds/:id`

Full update of a shed record, including grid position.

**Auth Required:** Yes

**Request Body:** Same shape as `POST /sheds` (all fields, without `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Shed updated successfully",
  "data": { /* updated Shed object */ }
}
```

---

### PATCH `/sheds/:id/position`

Update only the grid position of a shed. This is called when the user drags and repositions a shed in the grid layout.

**Auth Required:** Yes

**Request Body:**

```json
{
  "row": 2,
  "col": 1,
  "rowSpan": 1,
  "colSpan": 2
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Shed position updated",
  "data": {
    "id": "shed-uuid-001",
    "gridPosition": {
      "row": 2,
      "col": 1,
      "rowSpan": 1,
      "colSpan": 2
    }
  }
}
```

**Validation Rules:**
- `row`: required, integer >= 0
- `col`: required, integer >= 0
- `rowSpan`: required, integer >= 1
- `colSpan`: required, integer >= 1

**Frontend Impact:** Called on every drag-and-drop reposition. Must be fast. The frontend optimistically updates the UI and uses this endpoint to persist the change. If this endpoint is slow or fails, the grid will appear out of sync after page reload.

---

### DELETE `/sheds/:id`

Delete a shed and all associated daily records.

**Auth Required:** Yes

**Success Response — `204 No Content`**

---

## Daily Records Sub-Resource

---

### GET `/sheds/:id/daily-records`

List all daily records for a specific shed.

**Auth Required:** Yes

**Query Parameters:**

| Param   | Type    | Required | Description           |
|---------|---------|----------|-----------------------|
| `page`  | integer | No       | Page number           |
| `limit` | integer | No       | Page size, default 50 |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 15,
    "page": 1,
    "limit": 50,
    "data": [
      {
        "id": "record-uuid-001",
        "shedId": "shed-uuid-001",
        "type": "cleaning",
        "date": "2024-03-14",
        "details": "Full deep clean, disinfectant applied",
        "createdAt": "2024-03-14T14:00:00Z"
      }
    ]
  }
}
```

---

### POST `/sheds/:id/daily-records`

Add a daily activity record to a shed.

**Auth Required:** Yes

**Request Body:**

```json
{
  "type": "vaccination",
  "date": "2024-03-15",
  "details": "FMD vaccine administered to all 45 cows"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Daily record added",
  "data": {
    "id": "record-uuid-002",
    "shedId": "shed-uuid-001",
    "type": "vaccination",
    "date": "2024-03-15",
    "details": "FMD vaccine administered to all 45 cows",
    "createdAt": "2024-03-15T10:00:00Z"
  }
}
```

**Validation Rules:**
- `type`: required, one of: `feeding | vaccination | treatment | cleaning | mortality`
- `date`: required, valid ISO date
- `details`: required, 1–1000 characters

---

### DELETE `/sheds/:id/daily-records/:recordId`

Delete a specific daily record from a shed.

**Auth Required:** Yes

**Success Response — `204 No Content`**

---

## Notes

- The `animalType` field on a shed (`cow | sheep | chicken`) is separate from the `type` field on individual animals (`cattle | poultry | sheep | goat | pig | horse | other`). These are intentionally different enums — sheds use simplified types for grouping.
- `assignedWorker` is currently a free-form string. In future iterations, this could be a foreign key to the Workers module.
- The `productionData`, `feedSchedule`, `healthStatus`, and `cleaningStatus` fields are all free-form strings to allow flexibility. If structured data is needed later (e.g. production metrics as numbers), add new typed fields rather than changing existing ones to avoid breaking the frontend.
- Grid position conflicts (two sheds occupying the same cell) should ideally be validated server-side, but the frontend also enforces this visually.
