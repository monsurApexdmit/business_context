# Individual Livestock Management Module

**Frontend Route:** `/livestock`
**API Base Path:** `/livestock`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interfaces

```typescript
interface Animal {
  id: string;
  name: string;
  type: "cattle" | "poultry" | "sheep" | "goat" | "pig" | "horse" | "other";
  breed: string;
  tagId: string;              // Physical ear tag or RFID identifier
  dateOfBirth: string;        // ISO date string — "2022-05-10"
  weight: string;             // Free-form string, e.g. "450 kg"
  healthStatus: "healthy" | "sick" | "treatment" | "quarantine";
  location: string;           // Shed name, pasture name, etc.
  notes: string;
}

interface DailyRecord {
  id: string;
  animalId: string;
  type: "feeding" | "health" | "weight" | "medication" | "other";
  date: string;               // ISO date string
  details: string;
  createdAt: string;          // ISO datetime string
}

interface LivestockStats {
  total: number;
  healthy: number;
  sick: number;
  treatment: number;
  quarantine: number;
}
```

---

## Endpoints

---

### GET `/livestock`

List all individual animals for the active farm with optional filters and pagination.

**Auth Required:** Yes

**Query Parameters:**

| Param          | Type    | Required | Description                                               |
|----------------|---------|----------|-----------------------------------------------------------|
| `farm_id`      | string  | Yes      | Active farm UUID                                          |
| `search`       | string  | No       | Partial match on `name`, `tagId`, or `breed`              |
| `type`         | string  | No       | Filter by type: `cattle\|poultry\|sheep\|goat\|pig\|horse\|other` |
| `healthStatus` | string  | No       | Filter by: `healthy\|sick\|treatment\|quarantine`         |
| `page`         | integer | No       | Page number, default `1`                                  |
| `limit`        | integer | No       | Page size, default `20`                                   |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 154,
    "page": 1,
    "limit": 20,
    "data": [
      {
        "id": "animal-uuid-001",
        "name": "Bessie",
        "type": "cattle",
        "breed": "Angus",
        "tagId": "TAG-001",
        "dateOfBirth": "2021-03-15",
        "weight": "520 kg",
        "healthStatus": "healthy",
        "location": "North Pasture",
        "notes": "Lead cow, good milk producer"
      }
    ]
  }
}
```

---

### POST `/livestock`

Add a new animal record.

**Auth Required:** Yes

**Request Body:**

```json
{
  "name": "Bessie",
  "type": "cattle",
  "breed": "Angus",
  "tagId": "TAG-001",
  "dateOfBirth": "2021-03-15",
  "weight": "520 kg",
  "healthStatus": "healthy",
  "location": "North Pasture",
  "notes": "Lead cow",
  "farm_id": "farm-uuid-here"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Animal record created successfully",
  "data": { /* full Animal object with id */ }
}
```

**Validation Rules:**
- `name`: required, 1–100 characters
- `type`: required, must be one of the enum values
- `breed`: optional, max 100 characters
- `tagId`: optional, but if provided must be unique within the farm (return `409` on duplicate)
- `dateOfBirth`: optional, valid ISO date, must not be in the future
- `weight`: optional string
- `healthStatus`: required, one of: `healthy | sick | treatment | quarantine`
- `location`: optional string
- `notes`: optional string, max 1000 characters
- `farm_id`: required

---

### GET `/livestock/stats`

Returns count summary grouped by health status for the active farm.

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
    "total": 154,
    "healthy": 130,
    "sick": 10,
    "treatment": 8,
    "quarantine": 6
  }
}
```

**Important — Routing Note:** Register `/livestock/stats` **before** `/livestock/:id` in the router to prevent `"stats"` being parsed as an ID.

---

### GET `/livestock/:id`

Fetch a single animal by ID, including its daily records.

**Auth Required:** Yes

**Path Parameters:**

| Param | Type   | Description |
|-------|--------|-------------|
| `id`  | string | Animal UUID |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "id": "animal-uuid-001",
    "name": "Bessie",
    "type": "cattle",
    "breed": "Angus",
    "tagId": "TAG-001",
    "dateOfBirth": "2021-03-15",
    "weight": "520 kg",
    "healthStatus": "healthy",
    "location": "North Pasture",
    "notes": "Lead cow",
    "dailyRecords": [
      {
        "id": "record-uuid-001",
        "animalId": "animal-uuid-001",
        "type": "feeding",
        "date": "2024-03-15",
        "details": "Fed 5kg hay + 2kg grain",
        "createdAt": "2024-03-15T08:30:00Z"
      }
    ]
  }
}
```

**Note:** The `dailyRecords` array is embedded in the detail response for convenience. The frontend detail modal renders them inline.

---

### PUT `/livestock/:id`

Full update of an animal record.

**Auth Required:** Yes

**Request Body:** Same shape as `POST /livestock` (all fields, without `farm_id`).

**Success Response — `200 OK`:**

```json
{
  "message": "Animal updated successfully",
  "data": { /* updated Animal object */ }
}
```

---

### PATCH `/livestock/:id/health`

Update only the `healthStatus` field. Used by the health status quick-action on animal cards.

**Auth Required:** Yes

**Request Body:**

```json
{
  "healthStatus": "treatment"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Health status updated",
  "data": {
    "id": "animal-uuid-001",
    "healthStatus": "treatment"
  }
}
```

**Validation Rules:**
- `healthStatus`: required, must be one of: `healthy | sick | treatment | quarantine`

---

### DELETE `/livestock/:id`

Permanently delete an animal record and all associated daily records.

**Auth Required:** Yes

**Success Response — `204 No Content`**

**Error Response:**

| Code  | Scenario      |
|-------|---------------|
| `404` | Animal not found |

---

## Daily Records Sub-Resource

---

### GET `/livestock/:id/daily-records`

List all daily records for a specific animal.

**Auth Required:** Yes

**Query Parameters:**

| Param   | Type    | Required | Description          |
|---------|---------|----------|----------------------|
| `page`  | integer | No       | Page number          |
| `limit` | integer | No       | Page size, default 50 |

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "total": 30,
    "page": 1,
    "limit": 50,
    "data": [
      {
        "id": "record-uuid-001",
        "animalId": "animal-uuid-001",
        "type": "feeding",
        "date": "2024-03-15",
        "details": "Fed 5kg hay + 2kg grain",
        "createdAt": "2024-03-15T08:30:00Z"
      }
    ]
  }
}
```

---

### POST `/livestock/:id/daily-records`

Add a new daily record entry for a specific animal.

**Auth Required:** Yes

**Request Body:**

```json
{
  "type": "medication",
  "date": "2024-03-15",
  "details": "Administered 10ml antibiotic injection"
}
```

**Success Response — `201 Created`:**

```json
{
  "message": "Daily record added",
  "data": {
    "id": "record-uuid-002",
    "animalId": "animal-uuid-001",
    "type": "medication",
    "date": "2024-03-15",
    "details": "Administered 10ml antibiotic injection",
    "createdAt": "2024-03-15T09:00:00Z"
  }
}
```

**Validation Rules:**
- `type`: required, one of: `feeding | health | weight | medication | other`
- `date`: required, valid ISO date
- `details`: required, 1–1000 characters

**Note:** `createdAt` is set server-side. `animalId` is derived from the `:id` path parameter.

---

### DELETE `/livestock/:id/daily-records/:recordId`

Delete a specific daily record entry.

**Auth Required:** Yes

**Path Parameters:**

| Param      | Type   | Description       |
|------------|--------|-------------------|
| `id`       | string | Animal UUID       |
| `recordId` | string | Daily record UUID |

**Success Response — `204 No Content`**

**Error Responses:**

| Code  | Scenario                                           |
|-------|----------------------------------------------------|
| `404` | Record not found                                   |
| `403` | Record does not belong to the specified animal ID  |

---

## Notes

- `tagId` uniqueness should be scoped to the farm, not globally. Two farms can have `TAG-001`.
- `dateOfBirth` is used by the frontend to derive/display animal age. Ensure it is returned as a proper ISO date string.
- Daily records are append-only in the current UI. There is no edit-record endpoint. If editing is needed in the future, add `PUT /livestock/:id/daily-records/:recordId`.
- When an animal is deleted, cascade-delete all associated daily records in the database.
- The `GET /livestock/:id` detail view embeds `dailyRecords` for initial load. The separate `GET /livestock/:id/daily-records` endpoint is used for paginated loading if the animal has many records.
