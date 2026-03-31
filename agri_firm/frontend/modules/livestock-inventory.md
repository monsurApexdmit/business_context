# Livestock Inventory Module (by Type)

**Frontend Route:** `/livestock-inventory`
**API Base Path:** `/livestock-inventory`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## Purpose

This module provides an **aggregated, type-level view** of livestock across the farm. Unlike the Individual Livestock module (`/livestock`), which manages animals one by one, this module groups animals by species/type and shows aggregate health, capacity, and production metrics.

It is used for the inventory dashboard tab that shows summary cards per animal type (Cow, Sheep, Chicken) with drill-down into individual animals within each type.

---

## TypeScript Interfaces

```typescript
interface LivestockTypeData {
  type: "cow" | "sheep" | "chicken";
  label: string;                    // Display name, e.g. "Dairy Cows", "Sheep", "Broiler Chickens"
  healthyCount: number;
  sickCount: number;
  treatmentCount: number;
  quarantineCount: number;
  capacityUsed: number;             // Number of animals currently housed
  totalCapacity: number;            // Max capacity across all sheds for this type
  avgAge: string;                   // e.g. "2.4 years"
  avgWeight: string;                // e.g. "480 kg"
  productionRate: string;           // e.g. "95%", "120L/day" — free-form
}

interface AnimalDetail {
  id: string;
  type: "cow" | "sheep" | "chicken";
  age: string;                      // Derived or stored, e.g. "3 years 2 months"
  weight: string;                   // e.g. "500 kg"
  status: "healthy" | "sick" | "treatment" | "quarantine";
  condition: string;                // Free-form health condition note
  lastCheckup: string;              // ISO date string — "2024-03-10"
  notes: string;
}

interface LivestockInventorySummary {
  total: number;
  healthy: number;
  sick: number;
  treatment: number;
  quarantine: number;
}

interface LivestockInventoryResponse {
  types: LivestockTypeData[];
  summary: LivestockInventorySummary;
}
```

---

## Endpoints

---

### GET `/livestock-inventory`

Returns aggregated livestock data grouped by animal type, plus an overall summary.

**Auth Required:** Yes

**Query Parameters:**

| Param     | Type   | Required | Description                                   |
|-----------|--------|----------|-----------------------------------------------|
| `farm_id` | string | Yes      | Active farm UUID                              |
| `type`    | string | No       | Filter to a single type: `cow\|sheep\|chicken` |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "types": [
      {
        "type": "cow",
        "label": "Dairy Cows",
        "healthyCount": 82,
        "sickCount": 3,
        "treatmentCount": 2,
        "quarantineCount": 1,
        "capacityUsed": 88,
        "totalCapacity": 120,
        "avgAge": "3.1 years",
        "avgWeight": "490 kg",
        "productionRate": "118L/day"
      },
      {
        "type": "sheep",
        "label": "Sheep",
        "healthyCount": 145,
        "sickCount": 5,
        "treatmentCount": 0,
        "quarantineCount": 2,
        "capacityUsed": 152,
        "totalCapacity": 200,
        "avgAge": "1.8 years",
        "avgWeight": "62 kg",
        "productionRate": "92%"
      },
      {
        "type": "chicken",
        "label": "Broiler Chickens",
        "healthyCount": 980,
        "sickCount": 20,
        "treatmentCount": 0,
        "quarantineCount": 0,
        "capacityUsed": 1000,
        "totalCapacity": 1200,
        "avgAge": "6 weeks",
        "avgWeight": "2.1 kg",
        "productionRate": "88%"
      }
    ],
    "summary": {
      "total": 1240,
      "healthy": 1207,
      "sick": 28,
      "treatment": 2,
      "quarantine": 3
    }
  }
}
```

**Frontend Impact:**
- The `types` array drives the type summary cards. The frontend iterates over this array and renders one card per entry.
- The `summary` object populates the top-level stat bar.
- Adding or removing a type from the `types` array will add or remove a card dynamically.
- `label` is the display string shown on the card header. If missing or empty, the frontend falls back to capitalizing `type`.

**How aggregation works (backend guidance):**
- `healthyCount`, `sickCount`, etc. — aggregate from the individual animals table filtered by `type` and `farm_id`
- `capacityUsed` — sum of `totalAnimals` from all sheds where `animalType` matches
- `totalCapacity` — sum of `capacity` from all sheds where `animalType` matches
- `avgAge` — compute from `dateOfBirth` of all animals of that type; format as a human-readable string
- `avgWeight` — average of `weight` field (note: weight is currently a string in the Animal model — if numeric storage is needed, store separately)
- `productionRate` — free-form, can be left as a placeholder or computed from shed `productionData`

---

### GET `/livestock-inventory/:type/animals`

List individual animals for a specific livestock type with filtering and pagination.

**Auth Required:** Yes

**Path Parameters:**

| Param  | Type   | Description                          |
|--------|--------|--------------------------------------|
| `type` | string | Animal type: `cow \| sheep \| chicken` |

**Query Parameters:**

| Param     | Type    | Required | Description                                         |
|-----------|---------|----------|-----------------------------------------------------|
| `farm_id` | string  | Yes      | Active farm UUID                                    |
| `search`  | string  | No       | Partial match on `condition`, `notes`, or animal ID |
| `status`  | string  | No       | Filter: `healthy\|sick\|treatment\|quarantine`      |
| `page`    | integer | No       | Page number, default `1`                            |
| `limit`   | integer | No       | Page size, default `20`                             |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 88,
    "page": 1,
    "limit": 20,
    "data": [
      {
        "id": "animal-uuid-001",
        "type": "cow",
        "age": "3 years 2 months",
        "weight": "490 kg",
        "status": "healthy",
        "condition": "Excellent — regular checkup passed",
        "lastCheckup": "2024-03-10",
        "notes": "High milk producer"
      }
    ]
  }
}
```

**Note:** This endpoint is essentially a filtered view of the animals table. It returns `AnimalDetail` objects (the inventory-focused projection), not full `Animal` objects. The backend may either:
1. Query the `animals` table with a `type` filter and map to `AnimalDetail` shape, or
2. Maintain a separate `animal_inventory_view` or computed fields.

Option 1 is recommended for simplicity.

---

### PUT `/livestock-inventory/animals/:id`

Update an individual animal's inventory-relevant fields.

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description |
|-------|--------|-------------|
| `id`  | string | Animal UUID |

**Request Body:**

```json
{
  "age": "3 years 4 months",
  "weight": "505 kg",
  "status": "healthy",
  "lastCheckup": "2024-03-20",
  "notes": "Weight gain on track"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Animal record updated",
  "data": {
    "id": "animal-uuid-001",
    "type": "cow",
    "age": "3 years 4 months",
    "weight": "505 kg",
    "status": "healthy",
    "condition": "Excellent",
    "lastCheckup": "2024-03-20",
    "notes": "Weight gain on track"
  }
}
```

**Validation Rules:**
- `age`: optional string
- `weight`: optional string
- `status`: optional, must be one of: `healthy | sick | treatment | quarantine`
- `lastCheckup`: optional, valid ISO date
- `notes`: optional string, max 1000 characters

**Note:** This endpoint updates the same `animals` table record as `PUT /livestock/:id`. The difference is that it only exposes and updates the inventory-relevant fields. The backend should apply a partial update (only the fields provided in the request body).

---

## Notes

- This module's data is largely derived/aggregated from the same underlying animals and sheds data used by the Individual Livestock (`/livestock`) and Livestock Sheds (`/sheds`) modules. Ensure data consistency — changes made via `/livestock` or `/sheds` endpoints should be reflected when `/livestock-inventory` is re-fetched.
- The `type` filter in this module (`cow | sheep | chicken`) uses a simplified set compared to the Individual Livestock module (`cattle | poultry | sheep | goat | pig | horse | other`). The backend must map:
  - `cow` → animals where `type = "cattle"`
  - `sheep` → animals where `type = "sheep"`
  - `chicken` → animals where `type = "poultry"`
  This mapping should be documented and applied consistently.
- `condition` field on `AnimalDetail` is not present on the base `Animal` interface. It may be stored as an extra column on the animals table or derived from the most recent daily record of type `"health"`.
- `lastCheckup` on `AnimalDetail` may be derived from the most recent daily record of type `"health"` rather than stored separately.
