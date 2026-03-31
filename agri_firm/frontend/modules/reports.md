# Reports & Analytics Module

**Frontend Route:** `/reports`
**API Base Path:** `/reports`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## Overview

The Reports module provides aggregated, cross-module analytics data for charts and summary metrics. All endpoints are read-only. The data is computed by aggregating records from multiple modules (Finances, Crops, Livestock, Inventory, Schedule, Workers).

No CRUD operations exist in this module.

---

## TypeScript Interfaces

```typescript
interface RevenueDataPoint {
  month: string;        // "Jan", "Feb", ..., "Dec"
  revenue: number;      // Total income for the month
  expenses: number;     // Total expenses for the month
}

interface CropYieldDataPoint {
  crop: string;         // Crop name, e.g. "Corn", "Wheat"
  yield: number;        // Actual yield (numeric, in kg or tons — consistent unit)
  target: number;       // Target/estimated yield
}

interface LivestockDistributionDataPoint {
  type: string;         // Animal type, e.g. "Cattle", "Sheep", "Poultry"
  count: number;        // Total number of animals of this type
}

interface InventoryTrendDataPoint {
  month: string;        // "Jan", "Feb", ..., "Dec"
  seeds: number;        // Total seeds stock value or quantity for the month
  fertilizer: number;
  equipment: number;    // Count of equipment items
  feed: number;
}

interface WorkerProductivityDataPoint {
  month: string;        // "Jan", "Feb", ..., "Dec"
  tasksCompleted: number;
  hoursWorked: number;
}

interface ReportSummary {
  totalRevenue: number;
  netProfit: number;
  totalYield: string;   // Formatted string, e.g. "48,000 kg" — or numeric if consistent unit
  totalLivestock: number;
}
```

---

## Endpoints

---

### GET `/reports/summary`

Returns top-level key performance indicators for the farm dashboard or reports overview card.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                             |
|-----------|--------|----------|---------------------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                                        |
| `year`    | string | No       | Four-digit year, e.g. `"2024"`. Default: current year  |
| `period`  | string | No       | `monthly \| quarterly \| yearly`. Default: `yearly`      |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "totalRevenue": 245800,
    "netProfit": 98200,
    "totalYield": "48,000 kg",
    "totalLivestock": 1240
  }
}
```

**Data Sources:**
- `totalRevenue`: Sum of income transactions from Finances module for the period
- `netProfit`: `totalRevenue - totalExpenses`
- `totalYield`: Sum of `yieldEstimate` from Crops module (note: this field is a string in the current model — backend will need to parse/aggregate it or maintain a numeric field)
- `totalLivestock`: Count of all animals across the farm

**Frontend Impact:** This endpoint populates the summary stat row at the top of the Reports page. All four values are required.

---

### GET `/reports/revenue`

Monthly revenue vs. expenses breakdown for bar chart display.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                            |
|-----------|--------|----------|--------------------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                                       |
| `year`    | string | No       | Four-digit year, default: current year                 |
| `period`  | string | No       | `monthly \| quarterly \| yearly`. Default: `monthly`   |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    { "month": "Jan", "revenue": 14200,  "expenses": 8500  },
    { "month": "Feb", "revenue": 16800,  "expenses": 9200  },
    { "month": "Mar", "revenue": 21000,  "expenses": 11500 },
    { "month": "Apr", "revenue": 18400,  "expenses": 10800 },
    { "month": "May", "revenue": 22500,  "expenses": 12300 },
    { "month": "Jun", "revenue": 19800,  "expenses": 11000 },
    { "month": "Jul", "revenue": 25000,  "expenses": 14200 },
    { "month": "Aug", "revenue": 23600,  "expenses": 13500 },
    { "month": "Sep", "revenue": 20100,  "expenses": 11800 },
    { "month": "Oct", "revenue": 17400,  "expenses": 9900  },
    { "month": "Nov", "revenue": 15200,  "expenses": 8800  },
    { "month": "Dec", "revenue": 12000,  "expenses": 7500  }
  ]
}
```

**Important:** Always return all 12 months, even if some have zero activity (return `0` for those). Missing months break the chart's X-axis.

**Data Source:** Aggregated from the Finances module transactions table, grouped by month. This endpoint effectively mirrors `GET /finances/monthly` but uses `revenue` instead of `income` as the key name. Ensure consistency — consider aliasing or sharing the same underlying query.

---

### GET `/reports/crop-yield`

Actual vs. target yield per crop for bar/horizontal bar chart.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                           |
| `year`    | string | No       | Four-digit year, default: current year     |
| `season`  | string | No       | Filter by season: `spring\|summer\|fall\|winter` |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    { "crop": "Corn",    "yield": 4800,  "target": 5000  },
    { "crop": "Wheat",   "yield": 3200,  "target": 3000  },
    { "crop": "Barley",  "yield": 2100,  "target": 2500  },
    { "crop": "Soybean", "yield": 1800,  "target": 2000  }
  ]
}
```

**Data Source:** Aggregated from the Crops module. `yield` comes from `yieldEstimate` (parsed as numeric), `target` could be a separate `yieldTarget` field.

**Note on Current Data Model:** The Crop interface has `yieldEstimate` as a free-form string (e.g. `"4,800 kg"`). For this endpoint to work, the backend must either:
1. Parse the string and extract the numeric value, or
2. Add a `yieldTarget` numeric field and `yieldActual` numeric field to the crops table (recommended for future-proofing)

---

### GET `/reports/livestock-distribution`

Count of animals per type for pie/donut chart.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description      |
|-----------|--------|----------|------------------|
| `farm_id` | string | Yes      | Active farm UUID |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    { "type": "Cattle",  "count": 88  },
    { "type": "Sheep",   "count": 152 },
    { "type": "Poultry", "count": 1000 },
    { "type": "Goat",    "count": 45  },
    { "type": "Pig",     "count": 30  }
  ]
}
```

**Note:** Use display-friendly type names (Title Case: `"Cattle"`, not `"cattle"`). The frontend uses the `type` field as the pie chart legend label.

**Data Source:** `COUNT(*)` from the animals table grouped by `type` for the given farm.

---

### GET `/reports/inventory-trend`

Monthly stock level trends per inventory category for area/line chart.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                            |
|-----------|--------|----------|----------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                       |
| `year`    | string | No       | Four-digit year, default: current year |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    { "month": "Jan", "seeds": 120, "fertilizer": 80,  "equipment": 15, "feed": 200 },
    { "month": "Feb", "seeds": 95,  "fertilizer": 70,  "equipment": 15, "feed": 185 },
    { "month": "Mar", "seeds": 60,  "fertilizer": 90,  "equipment": 14, "feed": 170 },
    { "month": "Apr", "seeds": 30,  "fertilizer": 110, "equipment": 14, "feed": 155 },
    { "month": "May", "seeds": 80,  "fertilizer": 100, "equipment": 16, "feed": 195 },
    { "month": "Jun", "seeds": 75,  "fertilizer": 85,  "equipment": 16, "feed": 180 },
    { "month": "Jul", "seeds": 70,  "fertilizer": 75,  "equipment": 15, "feed": 160 },
    { "month": "Aug", "seeds": 65,  "fertilizer": 65,  "equipment": 15, "feed": 150 },
    { "month": "Sep", "seeds": 90,  "fertilizer": 70,  "equipment": 14, "feed": 175 },
    { "month": "Oct", "seeds": 110, "fertilizer": 80,  "equipment": 14, "feed": 190 },
    { "month": "Nov", "seeds": 100, "fertilizer": 85,  "equipment": 15, "feed": 185 },
    { "month": "Dec", "seeds": 115, "fertilizer": 90,  "equipment": 15, "feed": 200 }
  ]
}
```

**Note on Data Strategy:** Current inventory model only stores current quantities (no historical data). To support this trend endpoint, the backend must either:
1. Take a monthly snapshot of inventory quantities (store a `inventory_snapshots` table with month/year), or
2. Derive trends from inventory transaction history (restocking events)

Recommend option 1 with a scheduled task that snapshots inventory on the first day of each month.

**Always return all 12 months** with `0` for missing data.

---

### GET `/reports/worker-productivity`

Monthly task completion and hours worked per month for line chart.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                            |
|-----------|--------|----------|----------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                       |
| `year`    | string | No       | Four-digit year, default: current year |
| `period`  | string | No       | `monthly \| quarterly \| yearly`       |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": [
    { "month": "Jan", "tasksCompleted": 28, "hoursWorked": 480 },
    { "month": "Feb", "tasksCompleted": 31, "hoursWorked": 510 },
    { "month": "Mar", "tasksCompleted": 35, "hoursWorked": 560 },
    { "month": "Apr", "tasksCompleted": 30, "hoursWorked": 490 },
    { "month": "May", "tasksCompleted": 38, "hoursWorked": 620 },
    { "month": "Jun", "tasksCompleted": 33, "hoursWorked": 540 },
    { "month": "Jul", "tasksCompleted": 40, "hoursWorked": 650 },
    { "month": "Aug", "tasksCompleted": 37, "hoursWorked": 600 },
    { "month": "Sep", "tasksCompleted": 34, "hoursWorked": 550 },
    { "month": "Oct", "tasksCompleted": 29, "hoursWorked": 470 },
    { "month": "Nov", "tasksCompleted": 26, "hoursWorked": 430 },
    { "month": "Dec", "tasksCompleted": 22, "hoursWorked": 380 }
  ]
}
```

**Data Sources:**
- `tasksCompleted`: Count of tasks in the Schedule module where `status = 'Completed'` and `dueDate` falls within the month
- `hoursWorked`: The current Worker model does not have a `hoursWorked` field. This data requires either:
  1. A `hoursWorked` field on the Worker record (cumulative), or
  2. A time-tracking sub-resource (e.g. `POST /workers/:id/time-logs`)

  For an initial implementation, `hoursWorked` can be estimated as `tasksCompleted * averageTaskDuration` or returned as `0` until a time-tracking feature is added.

**Always return all 12 months** with `0` for months with no data.

---

## Notes

- All reports endpoints are **read-only** (GET only). No POST, PUT, or DELETE.
- All chart endpoints should return data sorted by month in chronological order (Jan → Dec), not alphabetically.
- The `period` query parameter on relevant endpoints allows the frontend to switch between monthly, quarterly, and yearly views. When `period=quarterly`, aggregate months into groups of 3 and use quarter labels (`"Q1"`, `"Q2"`, `"Q3"`, `"Q4"`) instead of month names.
- The `year` parameter defaults to the current year if not specified. Ensure the backend uses server-side current date for this default, not a hardcoded year.
- The `month` field in all response arrays must use exactly these three-letter abbreviations: `"Jan"`, `"Feb"`, `"Mar"`, `"Apr"`, `"May"`, `"Jun"`, `"Jul"`, `"Aug"`, `"Sep"`, `"Oct"`, `"Nov"`, `"Dec"`. The frontend chart library uses these as axis labels.
- Performance: These aggregation queries can be expensive. Consider adding database indexes on `date`, `farm_id`, `status`, and `type` fields in the source tables. For high-traffic scenarios, implement a caching layer with a TTL of 1 hour.
- A future enhancement could add a `GET /reports/export` endpoint that returns a pre-generated PDF or Excel report. This is not in scope for the current frontend but would be a natural addition.
