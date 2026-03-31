# Crop Management Module

**Frontend Route:** `/crops`
**API Base Path:** `/crops`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interface

```typescript
interface Crop {
  id: string;
  name: string;
  variety: string;
  fieldArea: string;
  plantedDate: string;        // ISO date string — "2024-03-15"
  expectedHarvest: string;    // ISO date string — "2024-08-20"
  status: "planted" | "growing" | "harvesting" | "harvested";
  season: "spring" | "summer" | "fall" | "winter";
  yieldEstimate: string;
  notes: string;
}

interface CropStats {
  total: number;
  planted: number;
  growing: number;
  harvesting: number;
  harvested: number;
}
```

---

## Endpoints

---

### GET `/crops`

List all crops for the authenticated farm, with optional filtering and pagination.

**Auth Required:** Yes

**Query Parameters:**

| Param      | Type    | Required | Description                                  |
|------------|---------|----------|----------------------------------------------|
| `farm_id`  | string  | Yes      | Active farm UUID                             |
| `search`   | string  | No       | Partial match on `name` or `variety`         |
| `status`   | string  | No       | Filter by status: `planted\|growing\|harvesting\|harvested` |
| `season`   | string  | No       | Filter by season: `spring\|summer\|fall\|winter` |
| `page`     | integer | No       | Page number, default `1`                     |
| `limit`    | integer | No       | Page size, default `20`                      |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 42,
    "page": 1,
    "limit": 20,
    "data": [
      {
        "id": "crop-uuid-001",
        "name": "Corn",
        "variety": "Sweet Yellow",
        "fieldArea": "12 acres",
        "plantedDate": "2024-04-01",
        "expectedHarvest": "2024-08-15",
        "status": "growing",
        "season": "summer",
        "yieldEstimate": "4,800 kg",
        "notes": "Irrigated twice weekly"
      }
    ]
  }
}
```

**Frontend Impact:** The crops list page reads `data.data` for the table rows and `data.total`, `data.page`, `data.limit` for pagination controls. Renaming any field breaks table rendering.

---

### POST `/crops`

Create a new crop record.

**Auth Required:** Yes

**Request Body:**

```json
{
  "name": "Corn",
  "variety": "Sweet Yellow",
  "fieldArea": "12 acres",
  "plantedDate": "2024-04-01",
  "expectedHarvest": "2024-08-15",
  "status": "planted",
  "season": "summer",
  "yieldEstimate": "4,800 kg",
  "notes": "Irrigated twice weekly",
  "farm_id": "farm-uuid-here"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Crop created successfully",
  "data": {
    "id": "crop-uuid-001",
    "name": "Corn",
    "variety": "Sweet Yellow",
    "fieldArea": "12 acres",
    "plantedDate": "2024-04-01",
    "expectedHarvest": "2024-08-15",
    "status": "planted",
    "season": "summer",
    "yieldEstimate": "4,800 kg",
    "notes": "Irrigated twice weekly"
  }
}
```

**Validation Rules:**
- `name`: required, 1–100 characters
- `variety`: optional, max 100 characters
- `fieldArea`: optional string (user-defined format, e.g. "12 acres")
- `plantedDate`: required, valid ISO date, must not be in the future for existing crops
- `expectedHarvest`: optional, valid ISO date, must be after `plantedDate`
- `status`: required, one of: `planted | growing | harvesting | harvested`
- `season`: required, one of: `spring | summer | fall | winter`
- `yieldEstimate`: optional string
- `notes`: optional string, max 1000 characters
- `farm_id`: required

---

### GET `/crops/stats`

Returns a count summary of crops grouped by status for the active farm.

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
    "total": 42,
    "planted": 10,
    "growing": 18,
    "harvesting": 8,
    "harvested": 6
  }
}
```

**Frontend Impact:** This endpoint is called when the crops page loads to populate the summary stat cards at the top of the page. Any field name change will silently show `0` on the dashboard cards.

**Important — Routing Note:** This endpoint path `/crops/stats` must be registered **before** `/crops/:id` in the backend router. Otherwise the string `"stats"` will be treated as a crop ID and return a `404`.

---

### GET `/crops/:id`

Fetch a single crop by ID.

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description  |
|-------|--------|--------------|
| `id`  | string | Crop UUID    |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "crop-uuid-001",
    "name": "Corn",
    "variety": "Sweet Yellow",
    "fieldArea": "12 acres",
    "plantedDate": "2024-04-01",
    "expectedHarvest": "2024-08-15",
    "status": "growing",
    "season": "summer",
    "yieldEstimate": "4,800 kg",
    "notes": "Irrigated twice weekly"
  }
}
```

**Error Response:**

| Code  | Scenario                    |
|-------|-----------------------------|
| `404` | Crop with given ID not found |

---

### PUT `/crops/:id`

Update all fields of an existing crop (full update).

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description |
|-------|--------|-------------|
| `id`  | string | Crop UUID   |

**Request Body:** Same shape as `POST /crops` (all fields, excluding `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Crop updated successfully",
  "data": { /* updated Crop object */ }
}
```

**Validation Rules:** Same as `POST /crops`.

---

### PATCH `/crops/:id/status`

Update only the status field of a crop. Used by the quick-action status dropdown on the crop list.

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description |
|-------|--------|-------------|
| `id`  | string | Crop UUID   |

**Request Body:**

```json
{
  "status": "harvesting"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Crop status updated",
  "data": {
    "id": "crop-uuid-001",
    "status": "harvesting"
  }
}
```

**Validation Rules:**
- `status`: required, must be one of: `planted | growing | harvesting | harvested`

**Frontend Impact:** The status badge on each crop card calls this endpoint on change. If this endpoint is removed, the frontend will fall back to `PUT /crops/:id` with the full object — ensure `PUT` is implemented first.

---

### DELETE `/crops/:id`

Delete a crop record permanently.

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description |
|-------|--------|-------------|
| `id`  | string | Crop UUID   |

**Success Response — `204 No Content`**

No response body.

**Error Response:**

| Code  | Scenario                    |
|-------|-----------------------------|
| `404` | Crop with given ID not found |

---

## CSV Export

CSV export is performed **client-side only**. The frontend uses the existing list data (fetched via `GET /crops`) and generates a CSV file in the browser using the `papaparse` or native CSV construction. **No backend endpoint is required for export.**

The export button triggers a download with the following columns:
`Name, Variety, Field Area, Planted Date, Expected Harvest, Status, Season, Yield Estimate, Notes`

---

## Notes

- The `status` field drives the color-coded badges on the crop list. The valid enum values must match exactly (case-sensitive).
- `plantedDate` and `expectedHarvest` are stored and returned as ISO date strings. The frontend formats them for display using `toLocaleDateString()`.
- `yieldEstimate` and `fieldArea` are free-form strings in the current model (not numeric), so no unit conversion is needed backend-side.
- If the backend implements soft-delete, ensure deleted crops do not appear in the list or stats endpoints.
