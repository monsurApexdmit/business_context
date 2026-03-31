# Field/Plot Map Module

**Frontend Route:** `/field-map`
**API Base Path:** `/fields`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interface

```typescript
interface FieldPlot {
  id: string;
  name: string;
  crop: string;               // Crop name or "Fallow" / "None"
  area: number;               // Numeric value in acres
  status: "growing" | "harvest-ready" | "fallow" | "planted";
  soilMoisture: number;       // Percentage: 0–100
  temperature: number;        // Celsius
  gridPosition: {
    row: number;              // 0-indexed grid row
    col: number;              // 0-indexed grid column
    rowSpan: number;          // Number of rows the plot spans
    colSpan: number;          // Number of columns the plot spans
  };
}

interface FieldStats {
  totalPlots: number;
  totalArea: number;          // Sum of area across all plots
  activeCrops: number;        // Count of plots with status "growing" or "planted"
  harvestReady: number;       // Count with status "harvest-ready"
  fallow: number;             // Count with status "fallow"
}
```

---

## Grid Layout Notes

The Field Map page renders field plots on a **6x6 CSS Grid** (6 rows × 6 columns). Each field plot occupies a rectangular area defined by its `gridPosition`.

- Grid coordinates are **0-indexed**: row 0, col 0 is top-left.
- `rowSpan` and `colSpan` define how many cells the plot occupies. Minimum value is `1` for both.
- Maximum values: `row + rowSpan <= 6` and `col + colSpan <= 6`.
- Plots can span multiple cells (e.g. a large field might span `rowSpan: 2, colSpan: 3`).
- The frontend uses CSS Grid `grid-row` and `grid-column` properties with the stored values.
- Grid position conflicts (overlapping plots) should be validated or warned at the API level.

---

## Endpoints

---

### GET `/fields`

List all field plots for the active farm.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description      |
|-----------|--------|----------|------------------|
| `farm_id` | string | Yes      | Active farm UUID |

**Note:** This endpoint does **not** use standard pagination — the grid renders all plots at once. All plots for the farm are returned in a single response.

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    {
      "id": "field-uuid-001",
      "name": "North Field",
      "crop": "Corn",
      "area": 12.5,
      "status": "growing",
      "soilMoisture": 68,
      "temperature": 24.3,
      "gridPosition": {
        "row": 0,
        "col": 0,
        "rowSpan": 2,
        "colSpan": 2
      }
    },
    {
      "id": "field-uuid-002",
      "name": "South Plot B",
      "crop": "Fallow",
      "area": 4.0,
      "status": "fallow",
      "soilMoisture": 30,
      "temperature": 26.1,
      "gridPosition": {
        "row": 0,
        "col": 2,
        "rowSpan": 1,
        "colSpan": 1
      }
    }
  ]
}
```

**Frontend Impact:** The array is rendered directly into the grid. Each item's `gridPosition` determines its CSS grid placement. If `gridPosition` is null or missing for any plot, that plot will not render.

---

### POST `/fields`

Create a new field plot.

**Auth Required:** Yes

**Request Body:**

```json
{
  "name": "North Field",
  "crop": "Corn",
  "area": 12.5,
  "status": "growing",
  "soilMoisture": 68,
  "temperature": 24.3,
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
  "message": "Field plot created successfully",
  "data": { /* full FieldPlot object with id */ }
}
```

**Validation Rules:**
- `name`: required, 1–100 characters, unique within the farm
- `crop`: optional string, max 100 characters (defaults to `"None"` or `"Fallow"` if empty)
- `area`: required, number > 0
- `status`: required, one of: `growing | harvest-ready | fallow | planted`
- `soilMoisture`: optional, number 0–100 (percentage)
- `temperature`: optional, number (Celsius, reasonable range: -20 to 60)
- `gridPosition.row`: required, integer >= 0, < 6
- `gridPosition.col`: required, integer >= 0, < 6
- `gridPosition.rowSpan`: required, integer >= 1; `row + rowSpan <= 6`
- `gridPosition.colSpan`: required, integer >= 1; `col + colSpan <= 6`
- `farm_id`: required

**Grid Conflict Check (recommended):** Validate that the new plot's grid cells do not overlap with any existing plot for the same farm. Return `409 Conflict` with a descriptive message if overlap is detected.

---

### GET `/fields/stats`

Returns aggregate statistics for all field plots in the active farm.

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
    "totalPlots": 12,
    "totalArea": 87.5,
    "activeCrops": 8,
    "harvestReady": 2,
    "fallow": 2
  }
}
```

**Field Definitions:**
- `totalPlots`: count of all field plots for the farm
- `totalArea`: `SUM(area)` across all plots
- `activeCrops`: count where `status IN ('growing', 'planted')`
- `harvestReady`: count where `status = 'harvest-ready'`
- `fallow`: count where `status = 'fallow'`

**Important — Routing Note:** Register `/fields/stats` **before** `/fields/:id`.

---

### GET `/fields/:id`

Fetch a single field plot by ID.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "field-uuid-001",
    "name": "North Field",
    "crop": "Corn",
    "area": 12.5,
    "status": "growing",
    "soilMoisture": 68,
    "temperature": 24.3,
    "gridPosition": {
      "row": 0,
      "col": 0,
      "rowSpan": 2,
      "colSpan": 2
    }
  }
}
```

---

### PUT `/fields/:id`

Full update of a field plot, including grid position.

**Auth Required:** Yes

**Request Body:** Same shape as `POST /fields` (all fields, without `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Field plot updated successfully",
  "data": { /* updated FieldPlot object */ }
}
```

---

### PATCH `/fields/:id/position`

Update only the grid position of a field plot. Called when the user drags a plot to a new position in the grid.

**Auth Required:** Yes

**Request Body:**

```json
{
  "row": 1,
  "col": 3,
  "rowSpan": 2,
  "colSpan": 2
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Field position updated",
  "data": {
    "id": "field-uuid-001",
    "gridPosition": {
      "row": 1,
      "col": 3,
      "rowSpan": 2,
      "colSpan": 2
    }
  }
}
```

**Validation Rules:**
- `row`: required, integer >= 0, < 6
- `col`: required, integer >= 0, < 6
- `rowSpan`: required, integer >= 1; `row + rowSpan <= 6`
- `colSpan`: required, integer >= 1; `col + colSpan <= 6`

**Grid Conflict Check (recommended):** Same as POST — verify that the new position does not overlap with another existing plot (excluding the current plot being moved).

**Frontend Impact:** Called on drag-and-drop completion. Should respond quickly. The frontend optimistically updates the UI and uses the response to confirm the change.

---

### DELETE `/fields/:id`

Delete a field plot.

**Auth Required:** Yes

**Success Response — `204 No Content`**

---

## Notes

- `soilMoisture` and `temperature` are numeric sensor readings. In the current model they are stored directly on the plot. In a future iteration, these could be moved to a time-series sensor readings table with `GET /fields/:id/readings` for historical data.
- The `crop` field is a free-form string (the crop name) rather than a foreign key to the Crops module. If relational linking is needed, add a `cropId` field separately.
- The grid is **6x6** in the current frontend implementation. If the grid size changes, the validation max values for `row`, `col`, `rowSpan`, and `colSpan` must be updated accordingly.
- `status` uses lowercase-with-hyphens format: `"harvest-ready"` (not `"Harvest Ready"` or `"harvestReady"`). This matches the CSS class names used for color-coding in the frontend.
- `area` is stored and returned as a `number` (float). The frontend appends `" acres"` for display purposes.
- When a new plot is created, suggest defaulting `soilMoisture` to `0` and `temperature` to `0` if not provided, rather than `null`, to avoid null-check errors in the frontend chart/display code.
